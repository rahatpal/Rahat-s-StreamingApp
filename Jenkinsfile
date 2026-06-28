pipeline {
    agent any

    environment {
        ECR_REGISTRY = '332779205001.dkr.ecr.us-east-1.amazonaws.com'
        REPO_FRONTEND = 'streamingapp-frontend'
        REPO_AUTH = 'streamingapp-auth'
        REPO_STREAMING = 'streamingapp-streaming'
        REPO_ADMIN = 'streamingapp-admin'
        REPO_CHAT = 'streamingapp-chat'
        IMAGE_TAG = env.BUILD_ID ?: 'latest'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/rahatpal/Rahat-s-StreamingApp.git'
            }
        }

        stage('Build Frontend') {
            steps {
                container('node') {
                    sh 'cd frontend && npm ci && npm run build'
                }
            }
        }

        stage('Build Backend Services') {
            steps {
                container('node') {
                    sh 'cd backend/authService && npm ci'
                    sh 'cd backend/streamingService && npm ci'
                    sh 'cd backend/adminService && npm ci'
                    sh 'cd backend/chatService && npm ci'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    // Build frontend
                    sh 'cd frontend && docker build -t ${env.ECR_REGISTRY}/${env.REPO_FRONTEND}:${env.IMAGE_TAG} .'
                    
                    // Build backend services
                    sh 'cd backend/authService && docker build -t ${env.ECR_REGISTRY}/${env.REPO_AUTH}:${env.IMAGE_TAG} .'
                    sh 'cd backend/streamingService && docker build -t ${env.ECR_REGISTRY}/${env.REPO_STREAMING}:${env.IMAGE_TAG} .'
                    sh 'cd backend/adminService && docker build -t ${env.ECR_REGISTRY}/${env.REPO_ADMIN}:${env.IMAGE_TAG} .'
                    sh 'cd backend/chatService && docker build -t ${env.ECR_REGISTRY}/${env.REPO_CHAT}:${env.IMAGE_TAG} .'
                }
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    def loginCommand = "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${env.ECR_REGISTRY}"
                    sh "${loginCommand}"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    // Push frontend
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_FRONTEND}:${env.IMAGE_TAG}'
                    sh 'docker tag ${env.ECR_REGISTRY}/${env.REPO_FRONTEND}:${env.IMAGE_TAG} ${env.ECR_REGISTRY}/${env.REPO_FRONTEND}:latest'
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_FRONTEND}:latest'
                    
                    // Push backend services
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_AUTH}:${env.IMAGE_TAG}'
                    sh 'docker tag ${env.ECR_REGISTRY}/${env.REPO_AUTH}:${env.IMAGE_TAG} ${env.ECR_REGISTRY}/${env.REPO_AUTH}:latest'
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_AUTH}:latest'
                    
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_STREAMING}:${env.IMAGE_TAG}'
                    sh 'docker tag ${env.ECR_REGISTRY}/${env.REPO_STREAMING}:${env.IMAGE_TAG} ${env.ECR_REGISTRY}/${env.REPO_STREAMING}:latest'
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_STREAMING}:latest'
                    
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_ADMIN}:${env.IMAGE_TAG}'
                    sh 'docker tag ${env.ECR_REGISTRY}/${env.REPO_ADMIN}:${env.IMAGE_TAG} ${env.ECR_REGISTRY}/${env.REPO_ADMIN}:latest'
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_ADMIN}:latest'
                    
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_CHAT}:${env.IMAGE_TAG}'
                    sh 'docker tag ${env.ECR_REGISTRY}/${env.REPO_CHAT}:${env.IMAGE_TAG} ${env.ECR_REGISTRY}/${env.REPO_CHAT}:latest'
                    sh 'docker push ${env.ECR_REGISTRY}/${env.REPO_CHAT}:latest'
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    // Create helm directory if missing
                    sh 'mkdir -p helm/charts/streamingapp'
                    
                    // Create basic Helm values.yaml if missing
                    sh '''cat > helm/charts/streamingapp/values.yaml << EOF
imageTag: ${env.IMAGE_TAG}
replicaCount: 1
frontend:
  servicePort: 80
  containerPort: 80
auth:
  servicePort: 3001
  containerPort: 3001
streaming:
  servicePort: 3002
  containerPort: 3002
admin:
  servicePort: 3003
  containerPort: 3003
chat:
  servicePort: 3004
  containerPort: 3004
EOF'''
                    
                    // Create basic Helm templates (minimal deployment)
                    sh '''mkdir -p helm/charts/streamingapp/templates
cat > helm/charts/streamingapp/templates/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: streamingapp-frontend
  namespace: streamingapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: streamingapp-frontend
  template:
    metadata:
      labels:
        app: streamingapp-frontend
    spec:
      containers:
      - name: frontend
        image: ${env.ECR_REGISTRY}/${env.REPO_FRONTEND}:${env.IMAGE_TAG}
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: streamingapp-auth
  namespace: streamingapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: streamingapp-auth
  template:
    metadata:
      labels:
        app: streamingapp-auth
    spec:
      containers:
      - name: auth
        image: ${env.ECR_REGISTRY}/${env.REPO_AUTH}:${env.IMAGE_TAG}
        ports:
        - containerPort: 3001
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: streamingapp-streaming
  namespace: streamingapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: streamingapp-streaming
  template:
    metadata:
      labels:
        app: streamingapp-streaming
    spec:
      containers:
      - name: streaming
        image: ${env.ECR_REGISTRY}/${env.REPO_STREAMING}:${env.IMAGE_TAG}
        ports:
        - containerPort: 3002
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: streamingapp-admin
  namespace: streamingapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: streamingapp-admin
  template:
    metadata:
      labels:
        app: streamingapp-admin
    spec:
      containers:
      - name: admin
        image: ${env.ECR_REGISTRY}/${env.REPO_ADMIN}:${env.IMAGE_TAG}
        ports:
        - containerPort: 3003
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: streamingapp-chat
  namespace: streamingapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: streamingapp-chat
  template:
    metadata:
      labels:
        app: streamingapp-chat
    spec:
      containers:
      - name: chat
        image: ${env.ECR_REGISTRY}/${env.REPO_CHAT}:${env.IMAGE_TAG}
        ports:
        - containerPort: 3004
EOF'''
                    
                    // Create service templates
                    sh '''cat > helm/charts/streamingapp/templates/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: streamingapp-frontend
  namespace: streamingapp
spec:
  selector:
    app: streamingapp-frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
---
apiVersion: v1
kind: Service
metadata:
  name: streamingapp-auth
  namespace: streamingapp
spec:
  selector:
    app: streamingapp-auth
  ports:
    - protocol: TCP
      port: 3001
      targetPort: 3001
---
apiVersion: v1
kind: Service
metadata:
  name: streamingapp-streaming
  namespace: streamingapp
spec:
  selector:
    app: streamingapp-streaming
  ports:
    - protocol: TCP
      port: 3002
      targetPort: 3002
---
apiVersion: v1
kind: Service
metadata:
  name: streamingapp-admin
  namespace: streamingapp
spec:
  selector:
    app: streamingapp-admin
  ports:
    - protocol: TCP
      port: 3003
      targetPort: 3003
---
apiVersion: v1
kind: Service
metadata:
  name: streamingapp-chat
  namespace: streamingapp
spec:
  selector:
    app: streamingapp-chat
  ports:
    - protocol: TCP
      port: 3004
      targetPort: 3004
EOF'''
                    
                    // Create namespace and deploy
                    sh 'kubectl create namespace streamingapp --dry-run=client -o yaml | kubectl apply -f -'
                    sh 'helm upgrade --install streamingapp helm/charts/streamingapp --namespace streamingapp --create-namespace'
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    sh 'kubectl rollout status deployment/streamingapp-frontend -n streamingapp'
                    sh 'kubectl rollout status deployment/streamingapp-auth -n streamingapp'
                    sh 'kubectl rollout status deployment/streamingapp-streaming -n streamingapp'
                    sh 'kubectl rollout status deployment/streamingapp-admin -n streamingapp'
                    sh 'kubectl rollout status deployment/streamingapp-chat -n streamingapp'
                    sh 'kubectl get svc -n streamingapp'
                }
            }
        }
    }

post {
    always {
        sh 'docker rmi ${env.ECR_REGISTRY}/${env.REPO_FRONTEND}:${env.IMAGE_TAG} || true'
        sh 'docker rmi ${env.ECR_REGISTRY}/${env.REPO_AUTH}:${env.IMAGE_TAG} || true'
        sh 'docker rmi ${env.ECR_REGISTRY}/${env.REPO_STREAMING}:${env.IMAGE_TAG} || true'
        sh 'docker rmi ${env.ECR_REGISTRY}/${env.REPO_ADMIN}:${env.IMAGE_TAG} || true'
        sh 'docker rmi ${env.ECR_REGISTRY}/${env.REPO_CHAT}:${env.IMAGE_TAG} || true'
    }
    success {
        echo "✅ Deployment successful! Frontend URL: \$(kubectl get svc streamingapp-frontend -n streamingapp -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
    }
    failure {
        echo "❌ Deployment failed!"
    }
}