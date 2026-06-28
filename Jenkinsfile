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
                sh 'docker build -t streamingapp-streaming-service ./backend/streamingService'
                sh 'docker build -t streamingapp-auth-service ./backend/authService'
                sh 'docker build -t streamingapp-admin-service ./backend/adminService'
                sh 'docker build -t streamingapp-chat-service ./backend/chatService'
                sh 'docker build -t streamingapp-frontend ./frontend'
            }
        }

        stage('Test') {
            steps {
                sh 'echo "Running basic validation..."'
                sh 'docker images'
            }
        }

        stage('Deploy') {
            steps {
                sh 'echo "Deploying application using Docker Compose..."'
                sh 'docker compose up -d'
            }
        }

    }

    post {

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }

        always {
            cleanWs()
        }
    }
}