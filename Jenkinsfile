pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE_NAME = 'streamingapp'
        DOCKER_TAG = 'latest'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/rahatpal/Rahat-s-StreamingApp.git'
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    // Build all services
                    sh 'docker build -t streamingapp-streaming-service ./backend/streamingService'
                    sh 'docker build -t streamingapp-auth-service ./backend/authService'
                    sh 'docker build -t streamingapp-admin-service ./backend/adminService'
                    sh 'docker build -t streamingapp-chat-service ./backend/chatService'
                    sh 'docker build -t streamingapp-frontend ./frontend'
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    // Run your tests here
                    sh 'echo "Running tests..."'
                    // Add your test commands here
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    // Deploy your application
                    sh 'echo "Deploying application..."'
                    // Add your deployment commands here
                }
            }
        }
    }
    
    post {
        always {
            cleanWs() // Clean up workspace after build
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
Step 5: Alternative Simpler Jenkinsfile
If you want to keep it simpler, here's a basic version:

pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/rahatpal/Rahat-s-StreamingApp.git'
            }
        }
        
        stage('Build') {
            steps {
                script {
                    sh 'echo "Building application..."'
                    // Add your build commands here
                    sh 'docker build -t streamingapp ./'
                }
            }
        }
        
        stage('Test') {
            steps {
                sh 'echo "Running tests..."'
                // Add test commands here
            }
        }
    }
}