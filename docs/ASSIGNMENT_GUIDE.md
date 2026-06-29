# Orchestration and Scaling Project — Rahat's StreamingApp

## Assignment Details
- **Jenkins URL**: https://jenkinsacademics.herovired.com/
- **Username**: herovired
- **Password**: herovired
- **Forked Repository**: https://github.com/rahatpal/Rahat-s-StreamingApp.git
- **Main Repository**: https://github.com/UnpredictablePrashant/StreamingApp.git
- **AWS Account**: 332779205001 (us-east-1)
- **EKS Cluster**: rahatstreamingapp-cluster
- **EC2 Management Node**: IP 172-31-17-156 (Amazon Linux 2)

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub Repository                        │
│              rahatpal/Rahat-s-StreamingApp (Fork)               │
└──────────────────────┬──────────────────────────────────────────┘
                       │ Push / Webhook
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Jenkins (HeroVired CI)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │Checkout  │→ │npm build │→ │ Docker   │→ │ Push to  │        │
│  │from Git  │  │Frontend  │  │ Build(×5)│  │ ECR      │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│                                              │                  │
│                                              ▼                  │
│                                    ┌──────────────────┐         │
│                                    │ Helm Deploy to   │         │
│                                    │ EKS              │         │
│                                    └──────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Amazon ECR (Image Registry)                    │
│  frontend │ auth-service │ streaming-service │ admin-service    │
│  chat-service                                                   │
└─────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              Amazon EKS (rahatstreamingapp-cluster)              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │frontend  │  │   auth   │  │streaming │  │  admin   │        │
│  │:80/LB    │  │:3001/CI  │  │:3002/CI  │  │:3003/CI  │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│  ┌──────────┐  ┌──────────┐                                     │
│  │   chat   │  │  mongo   │                                     │
│  │:3004/CI  │  │:27017    │                                     │
│  └──────────┘  └──────────┘                                     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Fluent Bit DaemonSet → CloudWatch Logs                  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ChatOps (Bonus Step)                          │
│  CloudWatch Alarm → SNS Topic → Lambda → Slack Webhook          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Version Control with Git

### 1.1 Fork the Repository
1. Go to https://github.com/UnpredictablePrashant/StreamingApp
2. Click Fork (top-right) select your GitHub account
3. Repository URL: https://github.com/rahatpal/Rahat-s-StreamingApp

### 1.2 Clone Your Fork
```bash
git clone https://github.com/rahatpal/Rahat-s-StreamingApp.git
cd Rahat-s-StreamingApp
```

### 1.3 Sync Fork with Upstream
```bash
git remote add upstream https://github.com/UnpredictablePrashant/StreamingApp.git
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

---

## Step 2: Prepare the MERN Application

### 2.1 Microservices Overview
| Service | Directory | Dockerfile | Port |
|---------|-----------|------------|------|
| Frontend | frontend/Dockerfile | Multi-stage (node:18 nginx) | 80 |
| Auth Service | backend/authService/Dockerfile | node:18-alpine | 3001 |
| Streaming Service | backend/streamingService/Dockerfile | node:18-alpine | 3002 |
| Admin Service | backend/adminService/Dockerfile | node:18-alpine | 3003 |
| Chat Service | backend/chatService/Dockerfile | node:18-alpine | 3004 |

### 2.2 Dockerfiles
Each service has a pre-existing Dockerfile. No modifications needed except:
- adminService: Ensure node:18-alpine and EXPOSE 3003

### 2.3 Create ECR Repositories
```bash
aws ecr create-repository --repository-name frontend --region us-east-1
aws ecr create-repository --repository-name auth-service --region us-east-1
aws ecr create-repository --repository-name streaming-service --region us-east-1
aws ecr create-repository --repository-name admin-service --region us-east-1
aws ecr create-repository --repository-name chat-service --region us-east-1
```

### 2.4 Build and Push Docker Images (Manual Test)
```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 332779205001.dkr.ecr.us-east-1.amazonaws.com

# Build and push frontend
docker build -t 332779205001.dkr.ecr.us-east-1.amazonaws.com/frontend:1 \
  --build-arg REACT_APP_AUTH_API_URL=http://auth-service:3001 \
  --build-arg REACT_APP_STREAMING_API_URL=http://streaming-service:3002 \
  --build-arg REACT_APP_ADMIN_API_URL=http://admin-service:3003 \
  --build-arg REACT_APP_CHAT_API_URL=http://chat-service:3004 \
  -f frontend/Dockerfile frontend/
docker push 332779205001.dkr.ecr.us-east-1.amazonaws.com/frontend:1

# Build and push backend services (repeat for each)
docker build -t 332779205001.dkr.ecr.us-east-1.amazonaws.com/auth-service:1 -f backend/authService/Dockerfile backend/authService/
docker push 332779205001.dkr.ecr.us-east-1.amazonaws.com/auth-service:1
```

---

## Step 3: AWS Environment Setup

### 3.1 Install AWS CLI (EC2 Management Node)
```bash
# Download and install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

# Verify
/usr/local/bin/aws --version
# aws-cli/2.35.11
```

### 3.2 Configure AWS Credentials
```bash
aws configure
# AWS Access Key ID: [your-access-key]
# AWS Secret Access Key: [your-secret-key]
# Default region: us-east-1
# Default output: json

# Verify
aws sts get-caller-identity
```

### 3.3 Install kubectl and eksctl
```bash
# kubectl v1.29 (EKS compatible)
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.6/2024-07-12/bin/linux/amd64/kubectl
chmod +x kubectl
mv kubectl /usr/local/bin/kubectl

# eksctl
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify all
kubectl version --client
eksctl version
helm version
```

### 3.4 Important: Use AWS CLI v2
The default /bin/aws is v1 (Python 2.7). Always use /usr/local/bin/aws:
```bash
export PATH=/usr/local/bin:$PATH
echo 'export PATH=/usr/local/bin:$PATH' >> ~/.bashrc
```

---

## Step 4: Continuous Integration (CI) using Jenkins

### 4.1 Access Jenkins
- URL: https://jenkinsacademics.herovired.com/
- Username: herovired
- Password: herovired

### 4.2 Add AWS Credentials to Jenkins
1. Manage Jenkins Credentials System Global credentials (unrestricted)
2. Click Add Credentials
3. Settings:
   - Kind: AWS Credentials
   - ID: Rahat aws-creds
   - Access Key: (your AWS access key)
   - Secret Key: (your AWS secret key)
4. Click Create

### 4.3 Create Jenkins Pipeline Job
1. New Item Name: Rahats-StreamingApp-Pipeline
2. Select Pipeline OK
3. Pipeline section:
   - Definition: Pipeline script from SCM
   - SCM: Git
   - Repository URL: https://github.com/rahatpal/Rahat-s-StreamingApp.git
   - Branch: */main
   - Script Path: Jenkinsfile
4. Save

### 4.4 Jenkinsfile (CI/CD Pipeline)
The Jenkinsfile at the root of the repository defines the complete pipeline with these stages:
1. Checkout - Pulls latest code from GitHub
2. Build Frontend - npm ci + CI=false npm run build
3. Build Docker Images - 5 parallel Docker builds (frontend, auth-service, streaming-service, admin-service, chat-service)
4. Push to ECR - Login and push all 5 images with build number tag
5. Deploy to EKS - kubectl config + Helm upgrade --install
6. Send SNS Alert - Publish success notification to rahat-streamingapp-alerts

### 4.5 Trigger Build
- Manual: Click Build Now in Jenkins
- Automatic: Jenkins is configured with GitHub webhook builds trigger on every push

### 4.6 Build Output Example (Build #8)
```
Checkout: Pulled commit "fix: helm service template, values, and EKS cluster name"
Build Frontend: npm ci + CI=false npm run build (191 kB JS)
Build Docker Images: 5 images built in parallel
Push to ECR: All images tagged :8 pushed
Deploy to EKS: Release "streamingapp" upgraded. Happy Helming!
Send SNS Alert: Published to rahat-streamingapp-alerts
```

---

## Step 5: Kubernetes Deployment (EKS)

### 5.1 Create EKS Cluster
Run from the EC2 management node:
```bash
eksctl create cluster \
    --name rahatstreamingapp-cluster \
    --region us-east-1 \
    --nodegroup-name standard-workers \
    --node-type t3.small \
    --nodes 2 \
    --nodes-min 1 \
    --nodes-max 3 \
    --managed
```
Time: ~15-20 minutes

### 5.2 Verify Cluster
```bash
aws eks update-kubeconfig --name rahatstreamingapp-cluster --region us-east-1
kubectl get nodes
```
Output:
```
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-0-86.ec2.internal     Ready    <none>   16m   v1.34.9-eks-93b80c6
ip-192-168-51-137.ec2.internal   Ready    <none>   16m   v1.34.9-eks-93b80c6
```

### 5.3 Helm Chart Structure
```
helm/charts/streamingapp/
  Chart.yaml          - apiVersion: v2, name: streamingapp
  values.yaml         - Configuration (image registry, ports, env vars)
  templates/
    deployment.yaml   - Pod definitions (6 deployments)
    service.yaml      - Network services (range over .Values.services)
```

### 5.4 Deploy via Helm
```bash
# From EC2 or Jenkins
helm upgrade --install streamingapp helm/charts/streamingapp \
    --namespace streamingapp --create-namespace \
    --set imageTag=8
```

### 5.5 Verify Deployment
```bash
kubectl get pods -n streamingapp
```
Output:
```
NAME                                   READY   STATUS    RESTARTS   AGE
admin-deployment-6cb7567748-fc9rt      1/1     Running   0          28m
auth-deployment-6cb7567748-fc9rt       1/1     Running   0          28m
chat-deployment-6cb7567748-fc9rt       1/1     Running   0          28m
frontend-deployment-6cb7567748-fc9rt   1/1     Running   0          28m
mongodb-deployment-6cb7567748-fc9rt    1/1     Running   0          28m
streaming-deployment-6cb7567748-fc9rt  1/1     Running   0          28m
```

```bash
kubectl get svc -n streamingapp
```
Output:
```
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP
admin-service       ClusterIP      10.100.58.248    <none>
auth-service        ClusterIP      10.100.168.138   <none>
chat-service        ClusterIP      10.100.213.253   <none>
frontend            LoadBalancer   10.100.235.227   a5b61d46f3ba2431b97f6a6cbadf7094-399125906.us-east-1.elb.amazonaws.com
streaming-service   ClusterIP      10.100.245.93    <none>
```

### 5.6 Test Application
Open the frontend LoadBalancer URL in browser:
http://a5b61d46f3ba2431b97f6a6cbadf7094-399125906.us-east-1.elb.amazonaws.com
Expected: StreamFlix React application displays and functions correctly.

### 5.7 Scale Pods (Scaling Requirement)
```bash
kubectl scale deployment frontend-deployment -n streamingapp --replicas=3
kubectl scale deployment auth-deployment -n streamingapp --replicas=2
kubectl get pods -n streamingapp
```

---

## Step 6: Monitoring and Logging (CloudWatch)

### 6.1 Create CloudWatch Log Group
```bash
aws logs create-log-group --log-group-name /aws/eks/rahatstreamingapp-cluster/fluentbit --region us-east-1
```

### 6.2 Attach IAM Policy to Node Role
```bash
aws iam attach-role-policy \
    --role-name eksctl-rahatstreamingapp-cluster-n-NodeInstanceRole-O8SXfBxLbfZT \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```
Wait 30 seconds for policy propagation.

### 6.3 Deploy Fluent Bit via Helm
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm upgrade --install fluent-bit eks/aws-for-fluent-bit \
    --namespace amazon-cloudwatch --create-namespace \
    --set cloudWatch.enabled=true \
    --set cloudWatch.region=us-east-1 \
    --set cloudWatch.logGroupName=/aws/eks/rahatstreamingapp-cluster/fluentbit
```

### 6.4 Verify Fluent Bit
```bash
kubectl get pods -n amazon-cloudwatch
```
Output:
```
NAME                                  READY   STATUS    RESTARTS   AGE
fluent-bit-aws-for-fluent-bit-6vhh6   1/1     Running   0          2m
fluent-bit-aws-for-fluent-bit-v7qhq   1/1     Running   0          2m
```

### 6.5 Verify CloudWatch Logs
```bash
aws logs describe-log-streams \
    --log-group-name /aws/eks/rahatstreamingapp-cluster/fluentbit \
    --region us-east-1 --query 'logStreams[].logStreamName'
```
Expected: Multiple log streams from all pods (frontend, backend services, MongoDB, kube-proxy).

---

## Bonus Step 9: ChatOps Integration

### 9.1 Create SNS Topic
```bash
aws sns create-topic --name rahat-streamingapp-alerts --region us-east-1
```
Topic ARN: arn:aws:sns:us-east-1:332779205001:rahat-streamingapp-alerts

### 9.2 Create Lambda IAM Role
```bash
cat > /tmp/lambda-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name rahat-slack-lambda-role \
    --assume-role-policy-document file:///tmp/lambda-trust.json

aws iam attach-role-policy --role-name rahat-slack-lambda-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### 9.3 Create Slack Webhook
1. Go to https://api.slack.com/apps
2. Create New App From scratch
3. Name: StreamingAppAlerts Select workspace Create App
4. Incoming Webhooks Activate
5. Add New Webhook to Workspace Select channel Allow
6. Copy webhook URL

### 9.4 Create Lambda Function
Lambda function code (Python 3.12) processes SNS events and forwards to Slack.
Runtime: python3.12, Handler: slack_notifier.lambda_handler

```bash
zip -j /tmp/slack_notifier.zip /tmp/slack_notifier.py

aws lambda create-function --function-name rahat-slack-notifier \
    --runtime python3.12 \
    --role arn:aws:iam::332779205001:role/rahat-slack-lambda-role \
    --handler slack_notifier.lambda_handler \
    --zip-file fileb:///tmp/slack_notifier.zip --region us-east-1
```

### 9.5 Subscribe Lambda to SNS
```bash
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:332779205001:rahat-streamingapp-alerts \
    --protocol lambda \
    --notification-endpoint arn:aws:lambda:us-east-1:332779205001:function:rahat-slack-notifier \
    --region us-east-1

aws lambda add-permission --function-name rahat-slack-notifier \
    --statement-id sns-trigger --action lambda:InvokeFunction \
    --principal sns.amazonaws.com \
    --source-arn arn:aws:sns:us-east-1:332779205001:rahat-streamingapp-alerts
```

### 9.6 Test ChatOps
```bash
aws sns publish \
    --topic-arn arn:aws:sns:us-east-1:332779205001:rahat-streamingapp-alerts \
    --message '{"AlarmName":"rahat-HighCPU","NewStateValue":"ALARM","NewStateReason":"CPU exceeded 80%"}' \
    --region us-east-1
```
Expected: Slack channel receives alert message with alarm name and status.

### 9.7 Jenkins SNS Integration
The Jenkins pipeline automatically publishes an SNS notification on successful deployment:
- Topic: rahat-streamingapp-alerts
- Message: "StreamingApp build #X deployed successfully to EKS"
- Subject: "StreamingApp CI/CD Success"
- Trigger: After successful Helm deployment stage

---

## Step 7: Documentation

### 7.1 Documentation Structure
All documentation is in the docs/ directory of the repository:
```
docs/
  ASSIGNMENT_GUIDE.md     - This file (complete guide)
  screenshots/            - Verification screenshots
```

### 7.2 Required Screenshots
| # | Screenshot | What to Capture |
|---|-----------|-----------------|
| 1 | Jenkins Build | Green checkmark for successful build |
| 2 | Jenkins Console | All 6 stages completed |
| 3 | ECR Repositories | 5 repos in AWS Console with tagged images |
| 4 | kubectl get pods | 6 pods in Running state |
| 5 | kubectl get svc | Services with frontend LoadBalancer URL |
| 6 | Frontend Browser | StreamFlix app loaded via ELB URL |
| 7 | CloudWatch Logs | Log group with fluentbit log streams |
| 8 | Slack Notification | Alert message received in Slack channel |

---

## Step 8: Final Validation

### 8.1 Verification Checklist
| Requirement | Status |
|------------|--------|
| GitHub fork created | ✅ |
| Dockerfiles exist for all services | ✅ |
| ECR repositories (5) created | ✅ |
| Jenkins pipeline runs successfully | ✅ |
| Docker images pushed to ECR | ✅ |
| EKS cluster with 2 nodes | ✅ |
| Helm deploys all 6 pods | ✅ |
| Frontend accessible via LoadBalancer | ✅ |
| Backend services reachable | ✅ |
| CloudWatch logging configured | ✅ |
| ChatOps (Slack) working | ✅ |
| Scaling works (kubectl scale) | ✅ |

### 8.2 Quick Validation Commands
```bash
# Pods
kubectl get pods -n streamingapp

# Services
kubectl get svc -n streamingapp

# Frontend URL
kubectl get svc frontend -n streamingapp -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Backend connectivity
kubectl exec -n streamingapp deploy/frontend-deployment -- wget -qO- http://auth-service:3001

# CloudWatch logs
aws logs describe-log-streams --log-group-name /aws/eks/rahatstreamingapp-cluster/fluentbit --region us-east-1

# Test SNS
aws sns publish --topic-arn arn:aws:sns:us-east-1:332779205001:rahat-streamingapp-alerts --message "Final validation" --region us-east-1
```

---

## Appendix: Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| kubectl fails with v1alpha1 error | aws-cli v1 generates old kubeconfig | Use /usr/local/bin/aws (v2) add to PATH |
| Helm: nil pointer evaluating | service.yaml wrong values path | Use range .Values.services pattern |
| Helm: namespace already exists | namespace.yaml template | Delete namespace.yaml from templates |
| Fluent Bit: AccessDeniedException | Node role missing permissions | Attach CloudWatchAgentServerPolicy |
| SNS: Topic does not exist | Wrong topic ARN in Jenkinsfile | Match ARN from aws sns list-topics |
| Frontend DNS: site unreachable | New ELB DNS propagation delay | Wait 2-5 minutes |
