# Orchestration and Scaling Project — Step-by-Step Guide

## Project Overview
This project implements a complete CI/CD pipeline for a MERN (MongoDB, Express.js, React, Node.js) application using Jenkins, Docker, AWS ECR, Kubernetes (EKS), monitoring tools (CloudWatch), and ChatOps (SNS + Slack).

## Architecture
```
GitHub (Source Code)
    │
    ▼
Jenkins (CI — HeroVired)
    │  ├── Checkout code
    │  ├── Build frontend (npm ci + npm run build)
    │  ├── Build Docker images (5 services)
    │  └── Push images to ECR
    │
    ▼
Amazon ECR (Docker Image Registry)
    │  ├── frontend
    │  ├── auth-service
    │  ├── streaming-service
    │  ├── admin-service
    │  └── chat-service
    │
    ▼
Amazon EKS (Kubernetes Cluster)
    │  ├── Deploy via Helm
    │  ├── Pods: frontend, auth, streaming, admin, chat, mongo
    │  └── Services: frontend (LoadBalancer), backend (ClusterIP)
    │
    ├── Monitoring: CloudWatch + Fluent Bit
    └── ChatOps: SNS → Lambda → Slack
```

---

## Step 1: Version Control with Git

### 1.1 Fork the Repository

1. Go to https://github.com/UnpredictablePrashant/StreamingApp
2. Click **Fork** (top-right) → select your GitHub account
3. Wait for fork to complete

### 1.2 Clone Your Fork Locally

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/StreamingApp.git
cd StreamingApp
```

### 1.3 Add Upstream Remote (for syncing updates)

```bash
git remote add upstream https://github.com/UnpredictablePrashant/StreamingApp.git
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

---

## Step 2: Prepare the MERN Application

The repository already contains all source code for 5 microservices. No code changes needed.

### 2.1 Microservices Overview

| Service | Directory | Port | Language/Framework |
|---------|-----------|------|-------------------|
| Frontend | `frontend/` | 80 (nginx) | React 18, MUI 5, Redux Toolkit |
| Auth Service | `backend/authService/` | 3001 | Node.js, Express, JWT |
| Streaming Service | `backend/streamingService/` | 3002 | Node.js, Express, S3 SDK |
| Admin Service | `backend/adminService/` | 3003 | Node.js, Express, Multer |
| Chat Service | `backend/chatService/` | 3004 | Node.js, Express, Socket.IO |
| MongoDB | (deployed separately) | 27017 | MongoDB 6.0 |

### 2.2 Dockerfiles

Each service has a pre-existing Dockerfile:

**Frontend** (`frontend/Dockerfile`) — Multi-stage build:
- Stage 1: `node:18-alpine` — installs dependencies + builds React app
- Stage 2: `nginx:1.27-alpine` — serves the built static files
- Build args for API URLs are passed at build time

**Backend Services** — Each has a single-stage Dockerfile:
- `node:18-alpine` base image
- `npm install --production` for dependencies
- Exposes the service's port
- Runs `npm start`

#### Verify adminService Dockerfile

Ensure `backend/adminService/Dockerfile` uses `node:18-alpine` (not `node:20-alpine`) and exposes port `3003` (not `3000`).

### 2.3 Frontend Build Configuration

The frontend `package.json` already includes:
```json
"scripts": {
    "build": "CI=false react-scripts build"
}
```
The `CI=false` flag prevents ESLint warnings from failing the build in CI.

### 2.4 Docker Compose for Local Testing (Optional)

```bash
docker-compose up --build
# Access frontend at http://localhost:3000
```

---

## Step 3: AWS Environment Setup

### 3.1 Install and Configure AWS CLI

```bash
# Verify AWS CLI is installed
aws --version

# Configure credentials
aws configure
# AWS Access Key ID: [your access key]
# AWS Secret Access Key: [your secret key]
# Default region: us-east-1
# Default output format: json

# Verify identity
aws sts get-caller-identity
```

**Expected output:**
```json
{
    "Account": "332779205001",
    "UserId": "AIDAU26ZPWGE...",
    "Arn": "arn:aws:iam::332779205001:user/Rahat-IAM"
}
```

### 3.2 Create ECR Repositories

```bash
# Create 5 ECR repositories (one per service)
aws ecr create-repository --repository-name frontend --region us-east-1
aws ecr create-repository --repository-name auth-service --region us-east-1
aws ecr create-repository --repository-name streaming-service --region us-east-1
aws ecr create-repository --repository-name admin-service --region us-east-1
aws ecr create-repository --repository-name chat-service --region us-east-1
```

### 3.3 Install Additional Tools

```powershell
# Install eksctl
# Download from: https://github.com/eksctl-io/eksctl/releases
# Verify: eksctl version

# Install Helm
# Download from: https://github.com/helm/helm/releases
# Verify: helm version

# Install kubectl
# Download from: https://kubernetes.io/docs/tasks/tools/
# Verify: kubectl version --client
```

---

## Step 4: Continuous Integration (CI) using Jenkins

### 4.1 Access Jenkins

- URL: https://jenkinsacademics.herovired.com/
- Username: `herovired`
- Password: `herovired`

### 4.2 Verify Installed Tools (on Jenkins)

The Jenkins environment has these tools pre-installed:
- Git ✅
- Docker ✅
- Node.js ✅
- npm ✅
- kubectl ✅
- Helm ✅
- AWS CLI ✅

### 4.3 Add AWS Credentials to Jenkins

1. Go to **Manage Jenkins** → **Credentials** → **System** → **Global credentials (unrestricted)**
2. Click **Add Credentials**
3. Fill in:

| Field | Value |
|-------|-------|
| Kind | **AWS Credentials** |
| ID | `aws-creds` |
| Description | `AWS Credentials for ECR/EKS` |
| Access Key ID | *(your AWS access key)* |
| Secret Access Key | *(your AWS secret key)* |

4. Click **Create**

### 4.4 Create the Jenkins Pipeline

1. Go to Jenkins Dashboard → **New Item**
2. Name: `Rahats-StreamingApp-Pipeline` (no apostrophe)
3. Select **Pipeline** → OK
4. Under **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/YOUR_USERNAME/StreamingApp.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
5. Click **Save**

### 4.5 Jenkinsfile

Create a file named `Jenkinsfile` in your repo root:

```groovy
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
                git branch: 'main', url: 'https://github.com/YOUR_USERNAME/StreamingApp.git'
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
                stage('Frontend') {
                    steps {
                        sh '''
                            docker build -t ${ECR_REGISTRY}/frontend:${IMAGE_TAG} \
                              --build-arg REACT_APP_AUTH_API_URL=http://auth-service:3001 \
                              --build-arg REACT_APP_STREAMING_API_URL=http://streaming-service:3002 \
                              --build-arg REACT_APP_ADMIN_API_URL=http://admin-service:3003 \
                              --build-arg REACT_APP_CHAT_API_URL=http://chat-service:3004 \
                              -f frontend/Dockerfile frontend/
                        '''
                    }
                }
                stage('Auth') {
                    steps {
                        sh 'docker build -t ${ECR_REGISTRY}/auth-service:${IMAGE_TAG} -f backend/authService/Dockerfile backend/authService/'
                    }
                }
                stage('Streaming') {
                    steps {
                        sh 'docker build -t ${ECR_REGISTRY}/streaming-service:${IMAGE_TAG} -f backend/streamingService/Dockerfile backend/streamingService/'
                    }
                }
                stage('Admin') {
                    steps {
                        sh 'docker build -t ${ECR_REGISTRY}/admin-service:${IMAGE_TAG} -f backend/adminService/Dockerfile backend/adminService/'
                    }
                }
                stage('Chat') {
                    steps {
                        sh 'docker build -t ${ECR_REGISTRY}/chat-service:${IMAGE_TAG} -f backend/chatService/Dockerfile backend/chatService/'
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([awsCredentials(credentialsId: 'aws-creds')]) {
                    sh '''
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
                          docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        docker push ${ECR_REGISTRY}/frontend:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/auth-service:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/streaming-service:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/admin-service:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/chat-service:${IMAGE_TAG}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed: Frontend built, all images pushed to ECR'
            echo '📌 Next: Deploy to EKS manually using Helm'
        }
        failure {
            echo '❌ Pipeline failed. Check logs above.'
        }
    }
}
```

> **Important**: Replace `YOUR_USERNAME` on line 10 with your actual GitHub username.

### 4.6 Push and Run

```bash
git add Jenkinsfile
git commit -m "Add Jenkins pipeline"
git push origin main
```

Then in Jenkins, click **Build Now**.

### 4.7 Pipeline Stages Explained

| Stage | Description | Time |
|-------|-------------|------|
| Checkout | Pulls latest code from GitHub | ~5s |
| Build Frontend | `npm ci` + `npm run build` | ~30s |
| Build Docker Images | 5 parallel Docker builds | ~2-3 min |
| Push to ECR | Login + push all 5 images | ~1 min |

### 4.8 Expected Jenkins Console Output

```
[Pipeline] stage
[Pipeline] { (Build Frontend)
...
> frontend@0.1.0 build
> CI=false react-scripts build
...
File sizes after gzip:
  191 kB  build/static/js/main.xxx.js
...
[Pipeline] stage
[Pipeline] { (Build Docker Images)
...
Successfully tagged 332779205001.dkr.ecr.us-east-1.amazonaws.com/frontend:5
...
[Pipeline] stage
[Pipeline] { (Push to ECR)
...
docker push 332779205001.dkr.ecr.us-east-1.amazonaws.com/frontend:5
...
✅ Pipeline completed: Frontend built, all images pushed to ECR
```

---

## Step 5: Kubernetes Deployment (EKS)

### 5.1 Create EKS Cluster

Run on your local machine (one-time setup):

```bash
eksctl create cluster \
    --name streamingapp-cluster \
    --region us-east-1 \
    --nodegroup-name standard-workers \
    --node-type t3.small \
    --nodes 2 \
    --nodes-min 1 \
    --nodes-max 3 \
    --managed
```

**Time**: ~15 minutes

**Verify**:
```bash
kubectl get nodes
```

### 5.2 Create Helm Chart

The Helm chart is located at `helm/charts/streamingapp/`.

**Chart structure**:
```
helm/charts/streamingapp/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Configuration values
└── templates/
    ├── namespace.yaml  # Kubernetes namespace
    ├── deployment.yaml # Pod definitions (all services + MongoDB)
    └── service.yaml    # Network services (LoadBalancer + ClusterIP)
```

### 5.3 Chart.yaml

```yaml
apiVersion: v2
name: streamingapp
description: A Helm chart for the StreamingApp microservices
type: application
version: 0.1.0
appVersion: "1.0"
```

### 5.4 values.yaml

```yaml
replicaCount: 1
namespace: streamingapp
imageRegistry: 332779205001.dkr.ecr.us-east-1.amazonaws.com
imageTag: latest

mongodb:
  image: mongo:6.0
  port: 27017

services:
  frontend:
    name: frontend
    image: frontend
    port: 80
    type: LoadBalancer
    env:
      REACT_APP_AUTH_API_URL: http://auth-service:3001
      REACT_APP_STREAMING_API_URL: http://streaming-service:3002
      REACT_APP_ADMIN_API_URL: http://admin-service:3003
      REACT_APP_CHAT_API_URL: http://chat-service:3004

  auth:
    name: auth
    image: auth-service
    port: 3001
    type: ClusterIP

  streaming:
    name: streaming
    image: streaming-service
    port: 3002
    type: ClusterIP

  admin:
    name: admin
    image: admin-service
    port: 3003
    type: ClusterIP

  chat:
    name: chat
    image: chat-service
    port: 3004
    type: ClusterIP
```

### 5.5 Deployment Template (`templates/deployment.yaml`)

```yaml
{{- range $key, $service := .Values.services }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $service.name }}-deployment
  namespace: {{ $.Values.namespace }}
spec:
  replicas: {{ $.Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ $service.name }}
  template:
    metadata:
      labels:
        app: {{ $service.name }}
    spec:
      containers:
        - name: {{ $service.name }}
          image: "{{ $.Values.imageRegistry }}/{{ $service.image }}:{{ $.Values.imageTag }}"
          ports:
            - containerPort: {{ $service.port }}
          {{- if $service.env }}
          env:
          {{- range $key, $value := $service.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
          {{- end }}
          {{- end }}
{{- end }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  namespace: {{ .Values.namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: "{{ .Values.mongodb.image }}"
          ports:
            - containerPort: {{ .Values.mongodb.port }}
```

### 5.6 Service Template (`templates/service.yaml`)

```yaml
{{- range $key, $service := .Values.services }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ $service.name }}-service
  namespace: {{ $.Values.namespace }}
spec:
  type: {{ $service.type }}
  selector:
    app: {{ $service.name }}
  ports:
    - port: {{ $service.port }}
      targetPort: {{ $service.port }}
{{- end }}
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: {{ .Values.namespace }}
spec:
  type: ClusterIP
  selector:
    app: mongodb
  ports:
    - port: {{ .Values.mongodb.port }}
      targetPort: {{ .Values.mongodb.port }}
```

### 5.7 Namespace Template (`templates/namespace.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace }}
```

### 5.8 Deploy to EKS

```bash
# Deploy with the build number from Jenkins as the image tag
helm upgrade --install streamingapp helm/charts/streamingapp \
    --namespace streamingapp --create-namespace \
    --set imageTag=BUILD_NUMBER

# Example: if Jenkins build #5
helm upgrade --install streamingapp helm/charts/streamingapp \
    --namespace streamingapp --create-namespace \
    --set imageTag=5
```

### 5.9 Verify Deployment

```bash
# Check pods
kubectl get pods -n streamingapp

# Expected output:
# NAME                                   READY   STATUS    RESTARTS   AGE
# admin-deployment-xxxxxxxxxx-xxxxx      1/1     Running   0          1m
# auth-deployment-xxxxxxxxxx-xxxxx       1/1     Running   0          1m
# chat-deployment-xxxxxxxxxx-xxxxx       1/1     Running   0          1m
# frontend-deployment-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
# mongodb-deployment-xxxxxxxxxx-xxxxx    1/1     Running   0          1m
# streaming-deployment-xxxxxxxxxx-xxxxx  1/1     Running   0          1m

# Check services
kubectl get svc -n streamingapp

# Expected output:
# NAME               TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
# admin-service      ClusterIP      10.100.xxx.xxx  <none>          3003/TCP
# auth-service       ClusterIP      10.100.xxx.xxx  <none>          3001/TCP
# chat-service       ClusterIP      10.100.xxx.xxx  <none>          3004/TCP
# frontend-service   LoadBalancer   10.100.xxx.xxx  abc.aws.com     80:xxxxx/TCP
# mongodb-service    ClusterIP      10.100.xxx.xxx  <none>          27017/TCP
# streaming-service  ClusterIP      10.100.xxx.xxx  <none>          3002/TCP

# Get frontend URL
kubectl get svc frontend-service -n streamingapp -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 5.10 Test Application

Open the frontend LoadBalancer URL in a browser. You should see the StreamingApp login page.

### 5.11 Scale Pods

```bash
# Scale frontend to 3 replicas
kubectl scale deployment frontend-deployment -n streamingapp --replicas=3

# Scale auth to 2 replicas
kubectl scale deployment auth-deployment -n streamingapp --replicas=2

# Verify
kubectl get pods -n streamingapp
```

---

## Step 6: Monitoring and Logging

### 6.1 Set Up CloudWatch with Fluent Bit

Create a namespace and service account:

```bash
kubectl create namespace amazon-cloudwatch
```

Create `fluent-bit-config.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: amazon-cloudwatch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-role
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit-role
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: amazon-cloudwatch
```

Create `fluent-bit-daemonset.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      k8s-app: fluent-bit
  template:
    metadata:
      labels:
        k8s-app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
        - name: fluent-bit
          image: amazon/aws-for-fluent-bit:latest
          env:
            - name: AWS_REGION
              value: us-east-1
            - name: CLUSTER_NAME
              value: streamingapp-cluster
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: fluent-bit-config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: fluent-bit-config
          configMap:
            name: fluent-bit-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: amazon-cloudwatch
data:
  fluent-bit.conf: |
    [SERVICE]
        Parsers_File parsers.conf
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Parser docker
        Tag kube.*
        Mem_Buf_Limit 50MB
        Skip_Long_Lines On
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
    [OUTPUT]
        Name cloudwatch_logs
        Match kube.*
        region us-east-1
        log_group_name /eks/streamingapp-cluster/containers
        log_stream_prefix streamingapp-
        auto_create_group true
```

Apply:

```bash
kubectl apply -f fluent-bit-config.yaml
kubectl apply -f fluent-bit-daemonset.yaml
```

### 6.2 Verify CloudWatch Logs

1. Go to AWS Console → CloudWatch → Log groups
2. Look for `/eks/streamingapp-cluster/containers`
3. Click on it to see log streams from your pods

### 6.3 Set Up CloudWatch Alarms

1. Go to AWS Console → CloudWatch → Alarms → Create alarm
2. Select metric → EKS → ContainerInsights → choose your cluster
3. Set condition (e.g., CPUUtilization > 80% for 5 minutes)
4. Add SNS notification (select the topic from Step 7)
5. Name the alarm and create

---

## Step 7: ChatOps Integration (Bonus)

### 7.1 Create SNS Topic

```bash
aws sns create-topic --name streamingapp-deploy-alerts --region us-east-1
```

Note the TopicArn:
```
arn:aws:sns:us-east-1:332779205001:streamingapp-deploy-alerts
```

### 7.2 Set Up Slack Webhook

1. Go to https://api.slack.com/apps
2. Click **Create New App** → **From scratch**
3. Name: `StreamingApp Alerts`, select your workspace
4. Go to **Incoming Webhooks** → **Activate Incoming Webhooks**
5. Click **Add New Webhook to Workspace**
6. Select a channel → **Allow**
7. Copy the Webhook URL

### 7.3 Create Lambda Function for SNS → Slack

1. Go to AWS Console → Lambda → **Create function**
2. Name: `sns-to-slack`
3. Runtime: Node.js 18.x
4. Click **Create function**

**Code** (`index.mjs`):

```javascript
import https from 'https';

export const handler = async (event) => {
    const message = event.Records[0].Sns.Message;
    const subject = event.Records[0].Sns.Subject || 'StreamingApp Alert';

    const slackWebhookUrl = 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL';

    const payload = JSON.stringify({
        text: `*${subject}*\n${message}`,
    });

    return new Promise((resolve, reject) => {
        const req = https.request(slackWebhookUrl, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
        }, (res) => {
            resolve('OK');
        });
        req.on('error', (e) => reject(e));
        req.write(payload);
        req.end();
    });
};
```

5. Add trigger: SNS → select `streamingapp-deploy-alerts`
6. Click **Add**

### 7.4 Test the Integration

```bash
aws sns publish \
    --topic-arn arn:aws:sns:us-east-1:332779205001:streamingapp-deploy-alerts \
    --message "✅ StreamingApp deployed successfully to EKS" \
    --subject "Deployment Success" \
    --region us-east-1
```

Check Slack for the alert message.

---

## Step 8: Documentation

### 8.1 Create Documentation

All documentation is stored in the `docs/` directory of your repository.

**Include**:
1. Architecture diagram (text-based or image)
2. Step-by-step setup guide (this document)
3. Configuration files (Jenkinsfile, Helm charts, Dockerfiles)
4. Screenshots of verification

### 8.2 Required Screenshots

| # | Screenshot | What to capture |
|---|-----------|-----------------|
| 1 | Jenkins pipeline | Green success status |
| 2 | Jenkins console | Build stages completed |
| 3 | ECR repositories | All 5 repos with images |
| 4 | `kubectl get pods` | All 6 pods in Running state |
| 5 | `kubectl get svc` | Services with LoadBalancer URL |
| 6 | Frontend in browser | Application accessible via URL |
| 7 | CloudWatch logs | Log streams for cluster |
| 8 | Slack alert | ChatOps notification received |

### 8.3 Push Documentation

```bash
git add docs/
git commit -m "Add project documentation with screenshots"
git push origin main
```

---

## Step 9: Final Validation

### 9.1 Checklist

| Requirement | Verification | Status |
|------------|-------------|--------|
| GitHub fork created | Repository visible in GitHub | ☐ |
| Dockerfiles for all services | `frontend/Dockerfile`, `backend/*/Dockerfile` | ☐ |
| ECR repositories created | 5 repos in AWS Console | ☐ |
| Jenkins pipeline runs | Green build in Jenkins | ☐ |
| Docker images pushed to ECR | Images visible in ECR repos | ☐ |
| EKS cluster created | `kubectl get nodes` shows nodes | ☐ |
| Helm chart deploys all services | `kubectl get pods -n streamingapp` shows 6 pods | ☐ |
| Frontend accessible | LoadBalancer URL works in browser | ☐ |
| CloudWatch logging configured | Log group exists in CloudWatch | ☐ |
| ChatOps integration working | Slack receives SNS alerts | ☐ |

### 9.2 Quick Verification Commands

```bash
# 1. Check pods
kubectl get pods -n streamingapp

# 2. Check services
kubectl get svc -n streamingapp

# 3. Check ECR images
aws ecr describe-repositories --region us-east-1
aws ecr list-images --repository-name frontend --region us-east-1

# 4. Check CloudWatch
aws logs describe-log-groups --region us-east-1

# 5. Test SNS
aws sns publish \
    --topic-arn arn:aws:sns:us-east-1:332779205001:streamingapp-deploy-alerts \
    --message "Final validation test" --region us-east-1
```

### 9.3 Submission

1. Push all code to your GitHub fork
2. Create a PDF or Word document containing your repository link
3. Repository link: `https://github.com/YOUR_USERNAME/StreamingApp`
4. Upload the file via Vlearn

---

## Troubleshooting

### Jenkins Build Fails

| Error | Cause | Solution |
|-------|-------|----------|
| `process apparently never started` | Apostrophe/special chars in job name | Create new job with plain name (no apostrophes) |
| `AccessDeniedException` when pushing to ECR | AWS credentials wrong or quarantined | Update Jenkins credentials with correct keys |
| `docker build` fails | Build context issue | Use correct `-f` path and context directory |
| `npm ci` fails | Network issue or package-lock missing | Run `npm install` instead of `npm ci` |

### Docker Build Issues

| Service | Build Context | Command |
|---------|--------------|---------|
| frontend | `frontend/` | `docker build -f frontend/Dockerfile frontend/` |
| authService | `backend/authService/` | `docker build -f backend/authService/Dockerfile backend/authService/` |
| streamingService | `backend/streamingService/` | `docker build -f backend/streamingService/Dockerfile backend/streamingService/` |
| adminService | `backend/adminService/` | `docker build -f backend/adminService/Dockerfile backend/adminService/` |
| chatService | `backend/chatService/` | `docker build -f backend/chatService/Dockerfile backend/chatService/` |

### EKS Deployment Issues

| Error | Cause | Solution |
|-------|-------|----------|
| `ImagePullBackOff` | Image tag doesn't exist in ECR | Check the image tag matches what's in ECR |
| `CrashLoopBackOff` | Missing env vars or DB connection | Check pod logs: `kubectl logs -n streamingapp pod-name` |
| `Pending` state | Insufficient cluster resources | Scale node group or reduce replica count |
| `ErrImagePull` | ECR permissions issue | Verify IAM role/node group has ECR pull permissions |
| `kubectl` not configured | Missing kubeconfig | Run `aws eks update-kubeconfig --name streamingapp-cluster --region us-east-1` |

### CloudWatch / Fluent Bit Issues

| Error | Cause | Solution |
|-------|-------|----------|
| No logs in CloudWatch | Fluent Bit not running | Check `kubectl get pods -n amazon-cloudwatch` |
| Fluent Bit pods in CrashLoop | Missing permissions | Ensure node group IAM role has CloudWatch logs permissions |

### ChatOps Issues

| Error | Cause | Solution |
|-------|-------|----------|
| No Slack message | Lambda not triggered by SNS | Check Lambda trigger configuration |
| Lambda execution error | Invalid Slack webhook URL | Verify webhook URL in Lambda code |
| SNS publish fails | Wrong topic ARN | Use correct TopicArn from `aws sns list-topics` |

---

## Quick Command Reference

### AWS
```bash
aws sts get-caller-identity
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 332779205001.dkr.ecr.us-east-1.amazonaws.com
aws eks update-kubeconfig --name streamingapp-cluster --region us-east-1
```

### Docker
```bash
docker build -t 332779205001.dkr.ecr.us-east-1.amazonaws.com/frontend:5 -f frontend/Dockerfile frontend/
docker push 332779205001.dkr.ecr.us-east-1.amazonaws.com/frontend:5
```

### Kubernetes
```bash
kubectl get pods -n streamingapp
kubectl get svc -n streamingapp
kubectl logs -n streamingapp deployment/frontend-deployment
kubectl scale deployment frontend-deployment -n streamingapp --replicas=3
kubectl delete pod -n streamingapp pod-name
```

### Helm
```bash
helm upgrade --install streamingapp helm/charts/streamingapp --namespace streamingapp --create-namespace --set imageTag=5
helm list -n streamingapp
helm uninstall streamingapp -n streamingapp
```

---

## Appendix: File Inventory

### Files You Need to Create/Modify

| File | Action | Notes |
|------|--------|-------|
| `Jenkinsfile` | **Create** | CI pipeline (checkout → build → push) |
| `backend/adminService/Dockerfile` | **Fix** | Change `node:20` → `node:18`, `EXPOSE 3000` → `EXPOSE 3003` |
| `helm/charts/streamingapp/values.yaml` | **Create/Replace** | Helm configuration with all 5 services |
| `helm/charts/streamingapp/templates/namespace.yaml` | **Create** | Define Kubernetes namespace |
| `helm/charts/streamingapp/templates/deployment.yaml` | **Create/Replace** | Deployments for all services + MongoDB |
| `helm/charts/streamingapp/templates/service.yaml` | **Create/Replace** | Services (frontend LoadBalancer + backend ClusterIP) |
| `docs/ASSIGNMENT_GUIDE.md` | **Create** | This documentation |
| `docs/screenshots/` | **Create** | Screenshots directory |

### Files Already Present (No Changes Needed)

| File | Purpose |
|------|---------|
| `frontend/Dockerfile` | Multi-stage frontend Docker build |
| `frontend/package.json` | React app with `CI=false` build script |
| `backend/authService/Dockerfile` | Auth service Docker build |
| `backend/streamingService/Dockerfile` | Streaming service Docker build |
| `backend/chatService/Dockerfile` | Chat service Docker build |
| `docker-compose.yml` | Local development orchestration |
| `helm/charts/streamingapp/Chart.yaml` | Helm chart metadata |
| `.gitignore` | Excludes node_modules, .env, etc. |
| `README.md` | Project README |
