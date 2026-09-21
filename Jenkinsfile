```groovy
pipeline {
    agent any

    environment {
        // Docker Hub repository
        IMAGE_NAME = "rahuldevops718/kanban-dashboard"

        // Jenkins credential ID
        DOCKER_CREDENTIALS = "dockerhub-credentials"

        // Docker container name
        CONTAINER_NAME = "kanban-dashboard"
    }

    stages {

        // ==========================================
        // 1. CHECKOUT
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
                }

                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
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
            }
        }


        // ==========================================
        // 4. PUSH TO DOCKER HUB
        // ==========================================
        stage('Push to Docker Hub') {
            steps {
                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                """

                echo "Docker image pushed successfully."
            }
        }


        // ==========================================
        // 5. DEPLOY NEW CONTAINER
        // ==========================================
        stage('Deploy New Container') {
            steps {
                script {

                    // Check if old container exists
                    def oldImage = sh(
                        script: """
                            docker inspect ${CONTAINER_NAME} \
                            --format='{{.Config.Image}}' 2>/dev/null || true
                        """,
                        returnStdout: true
                    ).trim()

                    // Save old image for rollback
                    if (oldImage) {
                        env.PREVIOUS_IMAGE = oldImage
                        echo "Previous working image: ${env.PREVIOUS_IMAGE}"
                    } else {
                        env.PREVIOUS_IMAGE = ""
                        echo "No previous container found. First deployment."
                    }

                    // Stop old container
                    sh """
                        docker stop ${CONTAINER_NAME} || true
                    """

                    // Remove old container
                    sh """
                        docker rm ${CONTAINER_NAME} || true
                    """

                    // Start new container
                    sh """
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p 5173:5173 \
```
