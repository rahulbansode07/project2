pipeline {
    agent any

    environment {

        // ==========================================
        // AWS ECR CONFIGURATION
        // ==========================================

        AWS_REGION = "ap-south-1"

        ECR_REGISTRY = "458563125461.dkr.ecr.ap-south-1.amazonaws.com"

        ECR_REPOSITORY = "458563125461.dkr.ecr.ap-south-1.amazonaws.com/rahul/project2"

        // Jenkins AWS credential ID
        AWS_CREDENTIALS = "aws-ecr-credentials"

        // Docker container name
        CONTAINER_NAME = "kanban-dashboard"

        // ==========================================
        // CONTAINER RESOURCE LIMITS
        // ==========================================

        // Maximum memory allowed
        MEMORY_LIMIT = "512m"

        // Maximum CPU allowed
        CPU_LIMIT = "0.5"
    }


    stages {

        // ==========================================
        // 1. CHECKOUT CODE
        // ==========================================

        stage('Checkout') {
            steps {

                git branch: 'main',
                    url: 'https://github.com/rahulbansode07/project2.git'
            }
        }


        // ==========================================
        // 2. DOCKER BUILD
        // ==========================================

        stage('Docker Build') {
            steps {

                script {

                    env.IMAGE_TAG = "build-${BUILD_NUMBER}"

                    echo "======================================"
                    echo "BUILDING DOCKER IMAGE"
                    echo "======================================"

                    echo "Image:"
                    echo "${ECR_REPOSITORY}:${IMAGE_TAG}"
                }


                sh """
                    docker build \
                    -t ${ECR_REPOSITORY}:${IMAGE_TAG} .
                """


                echo "Docker image built successfully."
            }
        }


        // ==========================================
        // 3. AWS ECR LOGIN
        // ==========================================

        stage('AWS ECR Login') {
            steps {

                echo "======================================"
                echo "LOGGING IN TO AWS ECR"
                echo "======================================"


                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS}"]
                ]) {

                    sh """
                        aws sts get-caller-identity

                        aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                        docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                    """
                }


                echo "AWS ECR login successful."
            }
        }


        // ==========================================
        // 4. PUSH IMAGE TO ECR
        // ==========================================

        stage('Push to ECR') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: "${AWS_CREDENTIALS}"]
                ]) {

                    sh """
                        docker push ${ECR_REPOSITORY}:${IMAGE_TAG}
                    """
                }


                echo "======================================"
                echo "IMAGE PUSHED TO ECR"
                echo "======================================"

                echo "Image:"
                echo "${ECR_REPOSITORY}:${IMAGE_TAG}"
            }
        }


        // ==========================================
        // 5. DEPLOY NEW CONTAINER
        // ==========================================

        stage('Deploy New Container') {
            steps {

                script {

                    echo "======================================"
                    echo "CHECKING CURRENT CONTAINER"
                    echo "======================================"


                    // Get currently running container image

                    def oldImage = sh(
                        script: """
                            docker inspect ${CONTAINER_NAME} \
                            --format='{{.Config.Image}}' \
                            2>/dev/null || true
                        """,
                        returnStdout: true
                    ).trim()


                    // Save previous image for rollback

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

                    echo "Memory Limit: ${MEMORY_LIMIT}"
                    echo "CPU Limit: ${CPU_LIMIT}"


                    sh """
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            --memory=${MEMORY_LIMIT} \
                            --cpus=${CPU_LIMIT} \
                            -p 5173:5173 \
                            ${ECR_REPOSITORY}:${IMAGE_TAG}
                    """


                    echo "======================================"
                    echo "NEW CONTAINER STARTED"
                    echo "======================================"

                    echo "Container : ${CONTAINER_NAME}"
                    echo "Image     : ${ECR_REPOSITORY}:${IMAGE_TAG}"
                    echo "Port      : 5173"
                    echo "Memory    : ${MEMORY_LIMIT}"
                    echo "CPU       : ${CPU_LIMIT}"

                    echo "======================================"
                }
            }
        }


        // ==========================================
        // 6. VERIFY RESOURCE LIMITS
        // ==========================================

        stage('Verify Resource Limits') {
            steps {

                echo "======================================"
                echo "VERIFYING CONTAINER RESOURCE LIMITS"
                echo "======================================"


                sh """
                    docker inspect ${CONTAINER_NAME} \
                    --format='Memory={{.HostConfig.Memory}} CPU={{.HostConfig.NanoCpus}}'
                """


                echo "Current container resource usage:"

                sh """
                    docker stats ${CONTAINER_NAME} \
                    --no-stream
                """


                echo "======================================"
                echo "RESOURCE LIMITS VERIFIED"
                echo "======================================"
            }
        }


        // ==========================================
        // 7. HEALTH CHECK
        // ==========================================

        stage('Health Check New Container') {
            steps {

                script {

                    echo "======================================"
                    echo "STARTING HEALTH CHECK"
                    echo "======================================"


                    def healthy = false


                    // Try 8 times
                    // 10 seconds between attempts

                    for (int i = 1; i <= 8; i++) {

                        def status = sh(
                            script: """
                                docker inspect \
                                --format='{{.State.Health.Status}}' \
                                ${CONTAINER_NAME}
                            """,
                            returnStdout: true
                        ).trim()


                        echo "Health check attempt ${i}/8"
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
                    // Health check failed
                    // ------------------------------------------

                    if (!healthy) {

                        echo "======================================"
                        echo "HEALTH CHECK FAILED"
                        echo "======================================"


                        echo "Container status:"

                        sh """
                            docker ps -a
                        """


                        echo "Container logs:"

                        sh """
                            docker logs ${CONTAINER_NAME} || true
                        """


                        error("New container failed health check.")
                    }


                    // ------------------------------------------
                    // HTTP check
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
            echo "Image        : ${ECR_REPOSITORY}:${IMAGE_TAG}"
            echo "Container    : ${CONTAINER_NAME}"
            echo "Port         : 5173"
            echo "Registry     : AWS ECR"
            echo "Region       : ${AWS_REGION}"
            echo "Memory Limit : ${MEMORY_LIMIT}"
            echo "CPU Limit    : ${CPU_LIMIT}"
            echo "Status       : SUCCESS"

            echo "======================================"
        }


        // ==========================================
        // FAILURE + ROLLBACK
        // ==========================================

        failure {

            echo "======================================"
            echo "        PIPELINE FAILED"
            echo "======================================"


            script {

                if (env.PREVIOUS_IMAGE?.trim()) {

                    echo "======================================"
                    echo "STARTING AUTOMATIC ROLLBACK"
                    echo "======================================"

                    echo "Previous image:"
                    echo "${env.PREVIOUS_IMAGE}"


                    // Stop failed container

                    sh """
                        docker stop ${CONTAINER_NAME} || true
                    """


                    // Remove failed container

                    sh """
                        docker rm ${CONTAINER_NAME} || true
                    """


                    // ------------------------------------------
                    // Start previous image WITH SAME LIMITS
                    // ------------------------------------------

                    echo "Starting previous working image..."

                    echo "Memory Limit: ${MEMORY_LIMIT}"
                    echo "CPU Limit: ${CPU_LIMIT}"


                    sh """
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            --memory=${MEMORY_LIMIT} \
                            --cpus=${CPU_LIMIT} \
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

                    echo "Memory Limit:"
                    echo "${MEMORY_LIMIT}"

                    echo "CPU Limit:"
                    echo "${CPU_LIMIT}"

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

            echo "Logging out from ECR..."

            sh """
                docker logout ${ECR_REGISTRY} || true
            """


            echo "Cleaning Jenkins workspace..."

            cleanWs()


            echo "Cleanup completed."
        }
    }
}