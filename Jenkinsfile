pipeline {
    agent any

    environment {
        // Change this to your Docker Hub username/repository
        IMAGE_NAME = "YOUR_DOCKERHUB_USERNAME/kanban-dashboard"

        // Jenkins Docker Hub credential ID
        DOCKER_CREDENTIALS = "dockerhub-credentials"
    }

    stages {

        // 1. Checkout code from GitHub
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 2. Build Docker image
        stage('Docker Build') {
            steps {
                script {
                    env.IMAGE_TAG = "build-${BUILD_NUMBER}"
                }

                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                """

                echo "Docker image built successfully."
            }
        }

        // 3. Login to Docker Hub
        stage('Docker Hub Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                        -u "$DOCKER_USERNAME" \
                        --password-stdin
                    '''
                }
            }
        }

        // 4. Push image to Docker Hub
        stage('Push to Docker Hub') {
            steps {
                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                """

                echo "Docker image pushed successfully."
            }
        }
    }

    post {
        success {
            echo "================================="
            echo "BUILD AND PUSH SUCCESSFUL"
            echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "================================="
        }

        failure {
            echo "================================="
            echo "BUILD OR PUSH FAILED"
            echo "================================="
        }

        always {
            sh 'docker logout || true'
            cleanWs()
        }
    }
}