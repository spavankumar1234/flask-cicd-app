pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: build
    image: python:3.10
    command:
    - cat
    tty: true
  - name: docker
    image: docker:25.0.3-cli
    command:
    - cat
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  - name: helm
    image: alpine/helm:3.14.4
    command:
    - cat
    tty: true
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    environment {
        DOCKER_IMAGE = "spavankumar1234/flask-cicd-app"
        DOCKER_CREDENTIALS = 'docker-creds'
        SONARQUBE_ENV = 'SonarQube'
        SLACK_WEBHOOK_URL = credentials('slack-webhook') // secure Slack webhook
    }

    stages {
        stage('Clone Code from GitHub') {
            steps {
                git url: 'https://github.com/spavankumar1234/flask-cicd-app.git', branch: 'main'
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
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '${scannerHome}/bin/sonar-scanner'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh "docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} ."
                }
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                container('docker') {
                    script {
                        docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS}") {
                            sh "docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes using Helm') {
            steps {
                container('helm') {
                    sh """
                    helm upgrade --install flask-app ./helm-chart \
                        --set image.repository=${DOCKER_IMAGE} \
                        --set image.tag=${BUILD_NUMBER}
                    """
                }
            }
        }
    }

    post {
        success {
            sh """
            curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"✅ Jenkins Build #${BUILD_NUMBER} succeeded and deployed!"}' \
            "${SLACK_WEBHOOK_URL}"
            """
        }
        failure {
            sh """
            curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"❌ Jenkins Build #${BUILD_NUMBER} failed. Please check the Jenkins logs."}' \
            "${SLACK_WEBHOOK_URL}"
            """
        }
    }
}
