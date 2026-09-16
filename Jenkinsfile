pipeline {
    agent any

    environment {
        // Docker Hub image
        IMAGE_NAME = "YOUR_DOCKERHUB_USERNAME/kanban-dashboard"

        // Application settings
        CONTAINER_NAME = "kanban-dashboard"
        APP_PORT = "5173"

        // Jenkins credentials
        DOCKER_CREDENTIALS = "dockerhub-credentials"
        SSH_CREDENTIALS = "ec2-ssh-key"

        // Deployment EC2
        DEPLOY_HOST = "YOUR_EC2_PUBLIC_IP"
        DEPLOY_USER = "ec2-user"
    }

    stages {

        // ------------------------------------------------
        // 1. CHECKOUT SOURCE
        // ------------------------------------------------
        stage('Checkout Source') {
            steps {
                checkout scm

                script {
                    env.GIT_SHA = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${BUILD_NUMBER}-${GIT_SHA}"

                    echo "Git SHA: ${GIT_SHA}"
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Docker Tag: ${IMAGE_TAG}"
                }
            }
        }


        // ------------------------------------------------
        // 2. DOCKER BUILD
        // ------------------------------------------------
        stage('Docker Build') {
            steps {
                sh """
                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} \
                    -t ${IMAGE_NAME}:build-${BUILD_NUMBER} \
                    .
                """

                sh "docker images ${IMAGE_NAME}"
            }
        }


        // ------------------------------------------------
        // 3. DOCKER REGISTRY LOGIN
        // ------------------------------------------------
        stage('Registry Login') {
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


        // ------------------------------------------------
        // 4. PUSH IMAGE
        // ------------------------------------------------
        stage('Push Docker Image') {
            steps {

                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:build-${BUILD_NUMBER}
                """
            }
        }


        // ------------------------------------------------
        // 5. DEPLOY TO EC2
        // ------------------------------------------------
        stage('Deploy to EC2') {
            steps {

                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {

                    sh """
                        ssh -o StrictHostKeyChecking=no \
                        ${DEPLOY_USER}@${DEPLOY_HOST} '

                            set -e

                            echo "Pulling new Docker image..."

                            docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                            echo "Stopping old container..."

                            docker stop ${CONTAINER_NAME} || true

                            docker rm ${CONTAINER_NAME} || true

                            echo "Starting new container..."

                            docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${APP_PORT}:${APP_PORT} \
                            ${IMAGE_NAME}:${IMAGE_TAG}

                            echo "Deployment completed."

                        '
                    """
                }
            }
        }


        // ------------------------------------------------
        // 6. HEALTH CHECK
        // ------------------------------------------------
        stage('Health Check') {
            steps {

                script {

                    sleep(time: 10, unit: 'SECONDS')

                    def result = sh(
                        script: """
                            curl -f --max-time 10 \
                            http://${DEPLOY_HOST}:${APP_PORT}/
                        """,
                        returnStatus: true
                    )

                    if (result != 0) {
                        error("Health check failed!")
                    }

                    echo "Health check passed."
                }
            }
        }
    }


    // ------------------------------------------------
    // SUCCESS / FAILURE / ROLLBACK
    // ------------------------------------------------

    post {

        success {
            echo "======================================"
            echo "DEPLOYMENT SUCCESSFUL"
            echo "Build: ${BUILD_NUMBER}"
            echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
            echo "======================================"
        }


        failure {
            echo "======================================"
            echo "DEPLOYMENT FAILED"
            echo "Starting rollback..."
            echo "======================================"

            script {

                sshagent(credentials: ["${SSH_CREDENTIALS}"]) {

                    sh """
                        ssh -o StrictHostKeyChecking=no \
                        ${DEPLOY_USER}@${DEPLOY_HOST} '

                            set +e

                            echo "Removing failed container..."

                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true

                            echo "Searching for previous image..."

                            PREVIOUS_IMAGE=\\\$(docker images \
                                ${IMAGE_NAME} \
                                --format "{{.Repository}}:{{.Tag}}" \
                                | grep -v "${IMAGE_TAG}" \
                                | head -1)

                            if [ -n "\\\$PREVIOUS_IMAGE" ]; then

                                echo "Rolling back to: \\\$PREVIOUS_IMAGE"

                                docker run -d \
                                --name ${CONTAINER_NAME} \
                                -p ${APP_PORT}:${APP_PORT} \
                                \\\$PREVIOUS_IMAGE

                            else

                                echo "No previous image found."
                                echo "Rollback could not be performed."

                            fi

                        '
                    """
                }
            }
        }


        always {
            echo "Cleaning Jenkins workspace..."

            sh """
                docker logout || true
            """

            cleanWs()
        }
    }
}
