pipeline {
    agent any

    environment {
        // Docker Hub repository
        IMAGE_NAME = "rahuldevops718/kanban-dashboard"

        // Jenkins credential ID
        DOCKER_CREDENTIALS = "dockerhub-credentials"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/rahulbansode07/project2.git'
            }
        }

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
            echo "======================================"
            echo "BUILD AND PUSH SUCCESSFUL"
            echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "======================================"
        }

        failure {
            echo "======================================"
            echo "BUILD OR PUSH FAILED"
            echo "======================================"
        }

        always {
            sh 'docker logout || true'
            cleanWs()
        }
    }
}