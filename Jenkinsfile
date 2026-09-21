pipeline {
    agent any

    environment {
        // Docker Hub repository
        IMAGE_NAME = "rahuldevops718/kanban-dashboard"

        // Jenkins Docker Hub credential ID
        DOCKER_CREDENTIALS = "dockerhub-credentials"

        // Docker container name
        CONTAINER_NAME = "kanban-dashboard"
    }

    stages {

        // ==========================================
        // 1. CHECKOUT CODE FROM GITHUB
        // ==========================================
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/rahulbansode07/project2.git'
            }
        }


        // ==========================================
        // 2. BUILD DOCKER IMAGE
        // ==========================================
        stage('Docker Build') {
            steps {
                script {

                    // Create unique image tag
                    env.IMAGE_TAG = "build-${BUILD_NUMBER}"

                    echo "Building image:"
                    echo "${IMAGE_NAME}:${IMAGE_TAG}"
                }

                sh """
                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} .
                """

                echo "Docker image built successfully."
            }
        }


        // ==========================================
        // 3. DOCKER HUB LOGIN
        // ==========================================
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

                echo "Docker Hub login successful."
            }
        }


        // ==========================================
        // 4. PUSH IMAGE TO DOCKER HUB
        // ==========================================
        stage('Push to Docker Hub') {
            steps {

                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                """

                echo "Docker image pushed successfully."
                echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }


        // ==========================================
        // 5. DEPLOY NEW CONTAINER
        // ==========================================
        stage('Deploy New Container') {
            steps {
                script {

                    echo "Checking current container..."

                    // Get currently running container image
                    def oldImage = sh(
                        script: """
                            docker inspect ${CONTAINER_NAME} \
                            --format='{{.Config.Image}}' \
                            2>/dev/null || true
                        """,
                        returnStdout: true
                    ).trim()


                    // ------------------------------------------
                    // Save previous image for rollback
                    // ------------------------------------------
                    if (oldImage) {

                        env.PREVIOUS_IMAGE = oldImage

                        echo "Previous image found:"
                        echo "${env.PREVIOUS_IMAGE}"

                    } else {

                        env.PREVIOUS_IMAGE = ""

                        echo "No previous container found."
                        echo "This is the first deployment."
                    }


                    // ------------------------------------------
                    // Stop old container
                    // ------------------------------------------
                    echo "Stopping old container..."

                    sh """
                        docker stop ${CONTAINER_NAME} || true
                    """


                    // ------------------------------------------
                    // Remove old container
                    // ------------------------------------------
                    echo "Removing old container..."

                    sh """
                        docker rm ${CONTAINER_NAME} || true
                    """


                    // ------------------------------------------
                    // Start new container
                    // ------------------------------------------
                    echo "Starting new container..."

                    sh """
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p 5173:5173 \
                            ${IMAGE_NAME}:${IMAGE_TAG}
                    """


                    echo "======================================"
                    echo "NEW CONTAINER STARTED"
                    echo "======================================"
                    echo "Container: ${CONTAINER_NAME}"
                    echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "Port: 5173"
                    echo "======================================"
                }
            }
        }


        // ==========================================
        // 6. HEALTH CHECK NEW CONTAINER
        // ==========================================
        stage('Health Check New Container') {
            steps {
                script {

                    echo "======================================"
                    echo "STARTING HEALTH CHECK"
                    echo "======================================"

                    def healthy = false

                    // Try 6 times
                    // 10 seconds between attempts
                    for (int i = 1; i <= 6; i++) {

                        def status = sh(
                            script: """
                                docker inspect \
                                --format='{{.State.Health.Status}}' \
                                ${CONTAINER_NAME}
                            """,
                            returnStdout: true
                        ).trim()


                        echo "Health check attempt ${i}/6"
                        echo "Docker Health Status: ${status}"


                        if (status == "healthy") {

                            healthy = true

                            echo "Container is HEALTHY."

                            break

                        } else if (status == "unhealthy") {

                            echo "Container is UNHEALTHY."

                            break

                        } else {

                            echo "Container is still starting..."

                            sleep 10
                        }
                    }


                    // ------------------------------------------
                    // If Docker health check failed
                    // ------------------------------------------
                    if (!healthy) {

                        echo "======================================"
                        echo "HEALTH CHECK FAILED"
                        echo "======================================"

                        echo "Container logs:"

                        sh """
                            docker logs ${CONTAINER_NAME} || true
                        """

                        error("New container failed health check.")
                    }


                    // ------------------------------------------
                    // Application HTTP check
                    // ------------------------------------------
                    echo "Checking application on port 5173..."

                    sh """
                        curl -f http://localhost:5173/ || exit 1
                    """


                    echo "======================================"
                    echo "HEALTH CHECK PASSED"
                    echo "APPLICATION IS HEALTHY"
                    echo "======================================"
                }
            }
        }
    }


    // ==========================================
    // POST ACTIONS
    // ==========================================
    post {

        // ==========================================
        // SUCCESS
        // ==========================================
        success {

            echo "======================================"
            echo "        PIPELINE SUCCESSFUL"
            echo "======================================"
            echo "Build Number : ${BUILD_NUMBER}"
            echo "Image        : ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "Container    : ${CONTAINER_NAME}"
            echo "Port         : 5173"
            echo "Status       : SUCCESS"
            echo "======================================"
        }


        // ==========================================
        // FAILURE + AUTOMATIC ROLLBACK
        // ==========================================
        failure {

            echo "======================================"
            echo "        PIPELINE FAILED"
            echo "======================================"

            echo "Checking whether rollback is possible..."


            script {

                if (env.PREVIOUS_IMAGE?.trim()) {

                    echo "======================================"
                    echo "STARTING AUTOMATIC ROLLBACK"
                    echo "======================================"

                    echo "Previous working image:"
                    echo "${env.PREVIOUS_IMAGE}"


                    // ------------------------------------------
                    // Stop failed/new container
                    // ------------------------------------------
                    sh """
                        docker stop ${CONTAINER_NAME} || true
                    """


                    // ------------------------------------------
                    // Remove failed/new container
                    // ------------------------------------------
                    sh """
                        docker rm ${CONTAINER_NAME} || true
                    """


                    // ------------------------------------------
                    // Start previous working image
                    // ------------------------------------------
                    echo "Starting previous working image..."

                    sh """
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p 5173:5173 \
                            ${env.PREVIOUS_IMAGE}
                    """


                    echo "Waiting for rollback container..."

                    sleep 15


                    // ------------------------------------------
                    // Verify rollback
                    // ------------------------------------------
                    echo "Checking rollback application..."

                    sh """
                        curl -f http://localhost:5173/ || exit 1
                    """


                    echo "======================================"
                    echo "        ROLLBACK SUCCESSFUL"
                    echo "======================================"
                    echo "Running image:"
                    echo "${env.PREVIOUS_IMAGE}"
                    echo "======================================"

                } else {

                    echo "======================================"
                    echo "NO PREVIOUS IMAGE FOUND"
                    echo "ROLLBACK NOT POSSIBLE"
                    echo "======================================"

                    echo "This was probably the first deployment."
                }
            }
        }


        // ==========================================
        // ALWAYS
        // ==========================================
        always {

            echo "Cleaning Docker login..."

            sh 'docker logout || true'

            echo "Cleaning Jenkins workspace..."

            cleanWs()

            echo "Cleanup completed."
        }
    }
}

