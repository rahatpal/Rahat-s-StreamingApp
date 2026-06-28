pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE_NAME = 'streamingapp'
        DOCKER_TAG = 'latest'
    }

    stages {

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -x

                    echo "========== CURRENT DIRECTORY =========="
                    pwd

                    echo "========== PROJECT FILES =========="
                    ls -la

                    echo "========== DOCKER VERSION =========="
                    docker --version

                    echo "========== DOCKER COMPOSE VERSION =========="
                    docker compose version || docker-compose --version

                    echo "========== BUILDING STREAMING SERVICE =========="
                    docker build --progress=plain -t streamingapp-streaming-service ./backend/streamingService

                    echo "========== BUILDING AUTH SERVICE =========="
                    docker build --progress=plain -t streamingapp-auth-service ./backend/authService

                    echo "========== BUILDING ADMIN SERVICE =========="
                    docker build --progress=plain -t streamingapp-admin-service ./backend/adminService

                    echo "========== BUILDING CHAT SERVICE =========="
                    docker build --progress=plain -t streamingapp-chat-service ./backend/chatService

                    echo "========== BUILDING FRONTEND =========="
                    docker build --progress=plain -t streamingapp-frontend ./frontend
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "========== AVAILABLE DOCKER IMAGES =========="
                    docker images
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "========== STARTING APPLICATION =========="
                    docker compose up -d || docker-compose up -d

                    echo "========== RUNNING CONTAINERS =========="
                    docker ps
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }

        failure {
            echo '❌ Pipeline failed!'
        }

        always {
            cleanWs()
        }
    }
}