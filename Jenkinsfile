pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Ada-Jesus/Boardgame-App.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('File System Scan (Trivy)') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qualityGate = waitForQualityGate credentialsId: '----'
                    if (qualityGate.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
                    }
                }
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-settings',
                    jdk: 'jdk17',
                    maven: 'maven3',
                    traceability: true
                ) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'mvn clean package'
                        sh 'docker build -t adajesus/boardgame:latest .'
                    }
                }
            }
        }

        stage('Docker Image Scan (Trivy)') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html adajesus/boardgame:latest'
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker push adajesus/boardgame:latest'
                    }
                }
            }
        }
        
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'kubernetes',
                    contextName: '',
                    credentialsId: 'k8-cred',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://172.31.88.199:6443'
                ) {
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'kubernetes',
                    contextName: '',
                    credentialsId: 'k8-cred',
                    namespace: 'webappss',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://172.31.88.199:6443'
                ) {
                    sh 'kubectl get pods -n webappss'
                    sh 'kubectl get svc -n webappss'
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName      = env.JOB_NAME
                def buildNumber  = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor  = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

                emailext(
                    subject: "${pipelineStatus}: Job '${jobName} [${buildNumber}]'",
                    body: """<p><strong>${pipelineStatus}</strong>: Job '${jobName} [${buildNumber}]'</p>
                             <p>Check console output at <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>""",
                    to: 'xiaominxia2000@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html'
                )
            }
            cleanWs()
        }
        success {
            echo "Pipeline succeeded! 🎉🎉"
        }
        failure {
            echo "Pipeline failed. ❌"
        }
        unstable {
            echo "Pipeline is unstable. ⚠️"
        }
        changed {
            echo "Pipeline status changed from the last run."
        }
    }
}
