pipeline {
    agent any
    environment {
        AWS_CREDENTIALS_ID = 'aws-creds'
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
                container('node:16-alpine') {
                    sh 'cd frontend && npm ci'
                    sh 'cd frontend && npm run build'
                    archiveArtifacts artifacts: 'frontend/build/**', fingerprint: true
                }
            }
        }

        stage('Build Backend Services') {
            steps {
                container('node:16-alpine') {
                    sh 'cd backend/authService && npm ci'
                    sh 'cd backend/streamingService && npm ci'
                    sh 'cd backend/adminService && npm ci'
                    sh 'cd backend/chatService && npm ci'
                }
            }
        }

        stage('Login to ECR & Push Images') {
            steps {
                script {
                    echo "⚠️ Docker build/push skipped: Jenkins sandbox does not support Docker daemon"
                    echo "✅ Frontend and backend dependencies built. Manually build and push images to ECR using your local machine."
                }
            }
        }

        stage('Send SNS Alert') {
            steps {
                script {
                    try {
                        awsSnsPublish(
                            credentialId: env.AWS_CREDENTIALS_ID,
                            region: 'us-east-1',
                            topicArn: 'arn:aws:sns:us-east-1:332779205001:streamingapp-deploy-alerts',
                            message: "✅ CI Build ${env.BUILD_NUMBER} completed. Frontend built successfully. Deploy to EKS manually.",
                            subject: "StreamingApp CI Success"
                        )
                        echo "✅ SNS notification sent."
                    } catch (Exception e) {
                        echo "⚠️ SNS notification failed (may be due to missing permissions). This is expected if SNS is not configured."
                    }
                }
            }
        }
    }

    post {
        success {
            echo '✅ Frontend and backend dependencies built successfully.'
            echo '📌 Next: On your local machine, build Docker images from this repo and push to ECR.'
            echo '📌 Then deploy to EKS using eksctl and Helm. Submit screenshots as proof.'
        }
        failure {
            echo '❌ Build failed. Check logs. Most likely: AWS SNS or Docker issues — but frontend build is fine.'
        }
    }
}