ipeline {
    agent any

    environment {
        AWS_REGION      = 'us-east-1'
        ECR_REGISTRY    = '332779205001.dkr.ecr.us-east-1.amazonaws.com'
        REPO_FRONTEND   = 'streamingapp-frontend'
        REPO_AUTH       = 'streamingapp-auth'
        REPO_STREAMING  = 'streamingapp-streaming'
        REPO_ADMIN      = 'streamingapp-admin'
        REPO_CHAT       = 'streamingapp-chat'
        IMAGE_TAG       = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/rahatpal/Rahat-s-StreamingApp.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('frontend') { sh 'npm ci' }
                dir('backend/authService') { sh 'npm ci' }
                dir('backend/streamingService') { sh 'npm ci' }
                dir('backend/adminService') { sh 'npm ci' }
                dir('backend/chatService') { sh 'npm ci' }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') { sh 'npm run build' }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh """
docker build -t ${ECR_REGISTRY}/${REPO_FRONTEND}:${IMAGE_TAG} frontend
docker build -t ${ECR_REGISTRY}/${REPO_AUTH}:${IMAGE_TAG} backend/authService
docker build -t ${ECR_REGISTRY}/${REPO_STREAMING}:${IMAGE_TAG} backend/streamingService
docker build -t ${ECR_REGISTRY}/${REPO_ADMIN}:${IMAGE_TAG} backend/adminService
docker build -t ${ECR_REGISTRY}/${REPO_CHAT}:${IMAGE_TAG} backend/chatService
"""
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                sh """
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
"""
            }
        }

        stage('Push Images') {
            steps {
                sh """
for IMG in ${REPO_FRONTEND} ${REPO_AUTH} ${REPO_STREAMING} ${REPO_ADMIN} ${REPO_CHAT}
do
  docker push ${ECR_REGISTRY}/$IMG:${IMAGE_TAG}
  docker tag ${ECR_REGISTRY}/$IMG:${IMAGE_TAG} ${ECR_REGISTRY}/$IMG:latest
  docker push ${ECR_REGISTRY}/$IMG:latest
done
"""
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
helm upgrade --install streamingapp helm/charts/streamingapp \
  --namespace streamingapp \
  --create-namespace \
  --set image.tag=${IMAGE_TAG}
"""
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
kubectl rollout status deployment/streamingapp-frontend -n streamingapp --timeout=180s
kubectl rollout status deployment/streamingapp-auth -n streamingapp --timeout=180s
kubectl rollout status deployment/streamingapp-streaming -n streamingapp --timeout=180s
kubectl rollout status deployment/streamingapp-admin -n streamingapp --timeout=180s
kubectl rollout status deployment/streamingapp-chat -n streamingapp --timeout=180s
kubectl get pods -n streamingapp
kubectl get svc -n streamingapp
"""
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
        }

        failure {
            echo 'Deployment failed.'
        }

        always {
            sh 'docker image prune -af || true'
        }
    }
}
