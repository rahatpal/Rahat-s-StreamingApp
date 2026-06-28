pipeline {
    agent any

    environment {
        ECR_REGISTRY = '332779205001.dkr.ecr.us-east-1.amazonaws.com'
        REPO_FRONTEND = 'streamingapp-frontend'
        REPO_AUTH = 'streamingapp-auth'
        REPO_STREAMING = 'streamingapp-streaming'
        REPO_ADMIN = 'streamingapp-admin'
        REPO_CHAT = 'streamingapp-chat'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/rahatpal/Rahat-s-StreamingApp.git'
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh '''
                        npm ci
                        npm run build
                    '''
                }
            }
        }

        stage('Build Backend Services') {
            steps {
                dir('backend/authService') { sh 'npm ci' }
                dir('backend/streamingService') { sh 'npm ci' }
                dir('backend/adminService') { sh 'npm ci' }
                dir('backend/chatService') { sh 'npm ci' }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh """cd frontend && docker build -t ${ECR_REGISTRY}/${REPO_FRONTEND}:${IMAGE_TAG} ."""
                    sh """cd backend/authService && docker build -t ${ECR_REGISTRY}/${REPO_AUTH}:${IMAGE_TAG} ."""
                    sh """cd backend/streamingService && docker build -t ${ECR_REGISTRY}/${REPO_STREAMING}:${IMAGE_TAG} ."""
                    sh """cd backend/adminService && docker build -t ${ECR_REGISTRY}/${REPO_ADMIN}:${IMAGE_TAG} ."""
                    sh """cd backend/chatService && docker build -t ${ECR_REGISTRY}/${REPO_CHAT}:${IMAGE_TAG} ."""
                }
            }
        }

        stage('Login to ECR') {
            steps {
                sh """aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_REGISTRY}"""
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh """docker push ${ECR_REGISTRY}/${REPO_FRONTEND}:${IMAGE_TAG}"""
                    sh """docker tag ${ECR_REGISTRY}/${REPO_FRONTEND}:${IMAGE_TAG} ${ECR_REGISTRY}/${REPO_FRONTEND}:latest"""
                    sh """docker push ${ECR_REGISTRY}/${REPO_FRONTEND}:latest"""

                    sh """docker push ${ECR_REGISTRY}/${REPO_AUTH}:${IMAGE_TAG}"""
                    sh """docker tag ${ECR_REGISTRY}/${REPO_AUTH}:${IMAGE_TAG} ${ECR_REGISTRY}/${REPO_AUTH}:latest"""
                    sh """docker push ${ECR_REGISTRY}/${REPO_AUTH}:latest"""

                    sh """docker push ${ECR_REGISTRY}/${REPO_STREAMING}:${IMAGE_TAG}"""
                    sh """docker tag ${ECR_REGISTRY}/${REPO_STREAMING}:${IMAGE_TAG} ${ECR_REGISTRY}/${REPO_STREAMING}:latest"""
                    sh """docker push ${ECR_REGISTRY}/${REPO_STREAMING}:latest"""

                    sh """docker push ${ECR_REGISTRY}/${REPO_ADMIN}:${IMAGE_TAG}"""
                    sh """docker tag ${ECR_REGISTRY}/${REPO_ADMIN}:${IMAGE_TAG} ${ECR_REGISTRY}/${REPO_ADMIN}:latest"""
                    sh """docker push ${ECR_REGISTRY}/${REPO_ADMIN}:latest"""

                    sh """docker push ${ECR_REGISTRY}/${REPO_CHAT}:${IMAGE_TAG}"""
                    sh """docker tag ${ECR_REGISTRY}/${REPO_CHAT}:${IMAGE_TAG} ${ECR_REGISTRY}/${REPO_CHAT}:latest"""
                    sh """docker push ${ECR_REGISTRY}/${REPO_CHAT}:latest"""
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    sh """helm upgrade --install streamingapp helm/charts/streamingapp \\
                        --namespace streamingapp \\
                        --create-namespace \\
                        --set imageTag=${IMAGE_TAG}"""
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    sh """kubectl rollout status deployment/streamingapp-frontend -n streamingapp --timeout=180s"""
                    sh """kubectl rollout status deployment/streamingapp-auth -n streamingapp --timeout=180s"""
                    sh """kubectl rollout status deployment/streamingapp-streaming -n streamingapp --timeout=180s"""
                    sh """kubectl rollout status deployment/streamingapp-admin -n streamingapp --timeout=180s"""
                    sh """kubectl rollout status deployment/streamingapp-chat -n streamingapp --timeout=180s"""
                    sh """kubectl get svc -n streamingapp"""
                }
            }
        }
    }

    post {
        always {
            sh 'docker image prune -af || true'
            sh 'docker system prune -af || true'
        }
        success {
            echo "✅ Deployment successful! Frontend URL: \$(kubectl get svc streamingapp-frontend -n streamingapp -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}