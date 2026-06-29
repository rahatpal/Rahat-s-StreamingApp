pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        ECR_REGISTRY = '332779205001.dkr.ecr.us-east-1.amazonaws.com'
        EKS_CLUSTER_NAME = 'streamingapp-cluster'
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
                    sh 'npm ci'
                    sh 'CI=false npm run build'
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Frontend Image') {
                    steps {
                        sh '''
                            docker build -t ${ECR_REGISTRY}/streamingapp-frontend:${IMAGE_TAG} \
                              --build-arg REACT_APP_AUTH_API_URL=http://auth-service:3001 \
                              --build-arg REACT_APP_STREAMING_API_URL=http://streaming-service:3002 \
                              --build-arg REACT_APP_ADMIN_API_URL=http://admin-service:3003 \
                              --build-arg REACT_APP_CHAT_API_URL=http://chat-service:3004 \
                              -f frontend/Dockerfile frontend/
                        '''
                    }
                }
                stage('Build Auth Image') {
                    steps {
                        sh 'docker build -t ${ECR_REGISTRY}/streamingapp-auth:${IMAGE_TAG} -f backend/authService/Dockerfile backend/authService/'
                    }
                }
                stage('Build Streaming Image') {
                    steps {
                        sh 'docker build -t ${ECR_REGISTRY}/streamingapp-streaming:${IMAGE_TAG} -f backend/streamingService/Dockerfile backend/streamingService/'
                    }
                }
                stage('Build Admin Image') {
                    steps {
                        sh 'docker build -t ${ECR_REGISTRY}/streamingapp-admin:${IMAGE_TAG} -f backend/adminService/Dockerfile backend/adminService/'
                    }
                }
                stage('Build Chat Image') {
                    steps {
                        sh 'docker build -t ${ECR_REGISTRY}/streamingapp-chat:${IMAGE_TAG} -f backend/chatService/Dockerfile backend/chatService/'
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
                          docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        docker push ${ECR_REGISTRY}/streamingapp-frontend:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/streamingapp-auth:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/streamingapp-streaming:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/streamingapp-admin:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/streamingapp-chat:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_DEFAULT_REGION}

                        helm upgrade --install streamingapp helm/charts/streamingapp \
                          --namespace streamingapp --create-namespace \
                          --set imageTag=${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Send SNS Alert') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws sns publish \
                          --topic-arn arn:aws:sns:us-east-1:332779205001:streamingapp-deploy-alerts \
                          --message "✅ StreamingApp build #${BUILD_NUMBER} deployed successfully to EKS" \
                          --subject "StreamingApp CI/CD Success"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed: Frontend built, images pushed to ECR, Helm deployed to EKS"
        }
        failure {
            echo "❌ Pipeline failed. Check logs above."
        }
    }
}