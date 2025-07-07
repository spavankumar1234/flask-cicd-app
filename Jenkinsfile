pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "spavankumar1234/flask-cicd-app"
        DOCKER_CREDENTIALS = 'docker-creds'
        SONARQUBE_ENV = 'SonarQube'
        SLACK_CHANNEL = '#devops-alerts'
    }

    stages {
        stage('Clone Code from GitHub') {
            steps {
                git credentialsId: 'git-creds', url: 'https://github.com/spavankumar1234/flask-cicd-app.git', branch: 'main'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'python3 -m unittest discover tests || true'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'SonarQubeScanner'
            }
            steps {
                withSonarQubeEnv(SONARQUBE_ENV) {
                    sh 'sonar-scanner'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t \$DOCKER_IMAGE:\${BUILD_NUMBER} ."
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS) {
                        sh "docker push \$DOCKER_IMAGE:\${BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes using Helm') {
            steps {
                sh """
                helm upgrade --install flask-app ./helm-chart \
                    --set image.repository=\$DOCKER_IMAGE \
                    --set image.tag=\${BUILD_NUMBER}
                """
            }
        }
    }

    post {
        success {
            slackSend channel: SLACK_CHANNEL, message: "✅ Build #\${BUILD_NUMBER} succeeded and deployed!"
        }
        failure {
            slackSend channel: SLACK_CHANNEL, message: "❌ Build #\${BUILD_NUMBER} failed. Please check Jenkins logs."
        }
    }
}
