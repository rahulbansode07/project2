pipeline {
    agent any

    environment {
        IMAGE_NAME = "rahuldevops718/kanban-dashboard"
        DOCKER_CREDENTIALS = "dockerhub-credentials"
        CONTAINER_NAME = "kanban-dashboard"
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

        stage('Deploy New Container') {
            steps {
                script {

                    echo "Checking previous container..."

                    def oldImage = sh(
                        script: """
                            docker inspect ${CONTAINER_NAME} \
                            --format='{{.Config.Image}}' 2>/dev/null || true
                        """,
                        returnStdout: true
                    ).trim()

                    if (oldImage) {
                        env.PREVIOUS_IMAGE = oldImage
                        echo "Previous image: ${env.PREVIOUS_IMAGE}"
                    } else {
                        env.PREVIOUS_IMAGE = ""
                        echo "No previous container found."
                    }

                    echo "Stopping old container..."

                    sh """
                        docker stop ${CONTAINER_NAME} || true
                    """

                    echo "Removing old container..."

                    sh """
                        docker rm ${CONTAINER_NAME} || true
                    """

                    echo "Starting new container..."

                    sh """
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p 5173:5173 \
                            ${IMAGE_NAME}:${IMAGE_TAG}
                    """

                    echo "New container started."
                }
            }
        }

        stage('Health Check New Container') {
            steps {
                script {

                    echo "Waiting for application to start..."

                    sleep 15

                    echo "Checking container status..."

                    sh """
                        docker ps \
                        --filter "name=${CONTAINER_NAME}" \
                        --filter "status=running" \
                        --format '{{.Names}}' | grep -w ${CONTAINER_NAME}
                    """

                    echo "Checking Docker health status..."

                    sh '''
                        STATUS=$(docker inspect \
                        --format='{{.State.Health.Status}}' \
                        kanban-dashboard)

                        echo "Health Status: $STATUS"

                        if [ "$STATUS" != "healthy" ]; then
                            echo "Health check failed."
                            exit 1
                        fi
                    '''

                    echo "Checking application..."

                    sh """
                        curl -f http://localhost:5173/ || exit 1
                    """

                    echo "======================================"
                    echo "HEALTH CHECK PASSED"
                    echo "======================================"
                }
            }
        }

        stage('Mark Success / Fail and Rollback') {
            steps {
                script {

                    try {

                        echo "Verifying deployment..."

                        sh """
                            docker ps \
                            --filter "name=${CONTAINER_NAME}" \
                            --filter "status=running" \
                            --format '{{.Names}}' | grep -w ${CONTAINER_NAME}
                        """

                        sh """
                            curl -f http://localhost:5173/ || exit 1
                        """

                        echo "======================================"
                        echo "DEPLOYMENT SUCCESSFUL"
                        echo "======================================"
                        echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                        echo "Container: ${CONTAINER_NAME}"
                        echo "======================================"

                    } catch (Exception e) {

                        echo "======================================"
                        echo "DEPLOYMENT FAILED"
                        echo "STARTING ROLLBACK"
                        echo "======================================"

                        if (env.PREVIOUS_IMAGE?.trim()) {

                            echo "Rolling back to:"
                            echo "${env.PREVIOUS_IMAGE}"

                            sh """
                                docker stop ${CONTAINER_NAME} || true
                                docker rm ${CONTAINER_NAME} || true

                                docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    -p 5173:5173 \
                                    ${env.PREVIOUS_IMAGE}
                            """

                            echo "Waiting for previous container..."

                            sleep 15

                            sh """
                                curl -f http://localhost:5173/ || exit 1
                            """

                            echo "======================================"
                            echo "ROLLBACK SUCCESSFUL"
                            echo "======================================"

                        } else {

                            echo "No previous image available."
                            echo "Rollback is not possible."
                        }

                        error("Deployment failed.")
                    }
                }
            }
        }
    }

    post {

        success {
            echo "======================================"
            echo "PIPELINE SUCCESSFUL"
            echo "======================================"
            echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "======================================"
        }

        failure {
            echo "======================================"
            echo "PIPELINE FAILED"
            echo "======================================"
        }

        always {
            sh 'docker logout || true'
            cleanWs()
        }
    }
}

