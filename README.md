# 🚀 End-to-End CI/CD Pipeline with Jenkins + AWS EKS + Monitoring

---

# 📌 Project Overview

This project demonstrates a complete production-style DevOps CI/CD workflow using:

* Jenkins
* Docker
* Amazon ECR
* Kubernetes
* AWS EKS
* Prometheus
* Grafana
* Horizontal Pod Autoscaler (HPA)

The pipeline automates:

✅ Docker Image Build
✅ Amazon ECR Push
✅ Kubernetes Deployment
✅ AWS EKS Release
✅ Monitoring Setup
✅ Auto Scaling

---

# 🏗️ Architecture

```text
Developer Push
      ↓
GitHub Repository
      ↓
Jenkins Pipeline
      ↓
Docker Build
      ↓
Amazon ECR
      ↓
AWS EKS Cluster
      ↓
Kubernetes Deployment
      ↓
Prometheus + Grafana
      ↓
HPA Auto Scaling
```

---

# 📋 Prerequisites

Install the following locally:

| Tool           | Purpose            |
| -------------- | ------------------ |
| AWS CLI        | AWS Authentication |
| kubectl        | Kubernetes CLI     |
| eksctl         | Create EKS Cluster |
| Docker Desktop | Docker Runtime     |
| Jenkins        | CI/CD Automation   |

---

# 🔐 Step 1 — Configure AWS CLI

```bash
aws configure
```

Provide:

* AWS Access Key
* AWS Secret Access Key
* Region → `us-east-1`
* Output → `json`

Verify:

```bash
aws sts get-caller-identity
```

---

# ☸️ Step 2 — Create AWS EKS Cluster

```bash
eksctl create cluster \
--name devops-cluster \
--region us-east-1 \
--zones us-east-1a,us-east-1b \
--nodegroup-name devops-nodes \
--node-type t3.small \
--nodes 1
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
STATUS = Ready
```

---

# 📦 Step 3 — Create Amazon ECR Repository

```bash
aws ecr create-repository \
--repository-name devops-app \
--region us-east-1
```

---

# 🔐 Step 4 — Login to Amazon ECR

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 336984083625.dkr.ecr.us-east-1.amazonaws.com
```

Expected:

```text
Login Succeeded
```

---

# 🖥️ Step 5 — Jenkins Access

Jenkins is already running locally using Docker Desktop.

Access Jenkins:

```text
http://localhost:8080
```

---

# 🔌 Step 6 — Install Jenkins Plugins

Go to:

```text
Manage Jenkins → Plugins
```

Install:

* Docker Pipeline
* Kubernetes CLI
* Git
* Pipeline
* Credentials Binding
* AWS Credentials
* Blue Ocean

Restart Jenkins if required.

---

# 🔑 Step 7 — Configure AWS Credentials in Jenkins

Go to:

```text
Manage Jenkins → Credentials → System → Global Credentials
```

Click:

```text
Add Credentials
```

Choose:

```text
AWS Credentials
```

Fill:

| Field       | Value                         |
| ----------- | ----------------------------- |
| Scope       | Global                        |
| ID          | eks-aws-creds                 |
| Description | AWS credentials for EKS CI/CD |

Provide:

* AWS Access Key ID
* AWS Secret Access Key

Click:

```text
Create
```

---

# 🐳 Step 8 — Verify Docker Access Inside Jenkins

Enter Jenkins container:

```bash
docker exec -it jenkins bash
```

Verify Docker:

```bash
docker ps
```

---

# ☸️ Step 9 — Install kubectl Inside Jenkins Container

Enter Jenkins container:

```bash
docker exec -it jenkins bash
```

Download kubectl:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Make executable:

```bash
chmod +x kubectl
```

Move to system path:

```bash
mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

# ☁️ Step 10 — Install AWS CLI Inside Jenkins Container

Inside Jenkins container:

```bash
apt update
apt install -y curl unzip
```

Download AWS CLI:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

Unzip:

```bash
unzip awscliv2.zip
```

Install:

```bash
./aws/install
```

Verify:

```bash
aws --version
```

---

# 🔐 Step 11 — Configure AWS CLI Inside Jenkins

Inside Jenkins container:

```bash
aws configure
```

Provide:

| Field          | Value           |
| -------------- | --------------- |
| AWS Access Key | Your AWS Key    |
| AWS Secret Key | Your AWS Secret |
| Region         | us-east-1       |
| Output         | json            |

Verify:

```bash
aws sts get-caller-identity
```

---

# ☸️ Step 12 — Configure EKS Access Inside Jenkins

Generate kubeconfig:

```bash
aws eks update-kubeconfig \
--region us-east-1 \
--name devops-cluster
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
STATUS = Ready
```

---

# 📂 Step 13 — Project Structure

```text
project/
│
├── app.py
├── requirements.txt
├── Dockerfile
├── Jenkinsfile
├── k8s-deployment.yaml
├── prometheus-deployment.yaml
├── grafana-deployment.yaml
├── hpa.yaml
└── README.md
```

---

# 🐳 Step 14 — Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

---

# ☸️ Step 15 — Kubernetes Deployment File

Create:

```text
k8s-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: devops-app

spec:
  replicas: 2

  selector:
    matchLabels:
      app: devops-app

  template:
    metadata:
      labels:
        app: devops-app

    spec:
      containers:
      - name: devops-app

        image: 336984083625.dkr.ecr.us-east-1.amazonaws.com/devops-app:latest

        ports:
        - containerPort: 5000

        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"

          limits:
            cpu: "500m"
            memory: "512Mi"

---

apiVersion: v1
kind: Service

metadata:
  name: devops-service

spec:
  selector:
    app: devops-app

  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000

  type: LoadBalancer
```

---

# ⚖️ Step 16 — Horizontal Pod Autoscaler

Create:

```text
hpa.yaml
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: devops-hpa

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: devops-app

  minReplicas: 1
  maxReplicas: 5

  metrics:
  - type: Resource

    resource:
      name: cpu

      target:
        type: Utilization
        averageUtilization: 50
```

Apply:

```bash
kubectl apply -f hpa.yaml
```

Verify:

```bash
kubectl get hpa
```

---

# 📊 Step 17 — Deploy Prometheus

Create:

```text
prometheus-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: prometheus

spec:
  replicas: 1

  selector:
    matchLabels:
      app: prometheus

  template:
    metadata:
      labels:
        app: prometheus

    spec:
      containers:
      - name: prometheus

        image: prom/prometheus

        ports:
        - containerPort: 9090

---

apiVersion: v1
kind: Service

metadata:
  name: prometheus-service

spec:
  selector:
    app: prometheus

  ports:
    - port: 9090
      targetPort: 9090

  type: LoadBalancer
```

Deploy:

```bash
kubectl apply -f prometheus-deployment.yaml
```

---

# 📈 Step 18 — Deploy Grafana

Create:

```text
grafana-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: grafana

spec:
  replicas: 1

  selector:
    matchLabels:
      app: grafana

  template:
    metadata:
      labels:
        app: grafana

    spec:
      containers:
      - name: grafana

        image: grafana/grafana

        ports:
        - containerPort: 3000

---

apiVersion: v1
kind: Service

metadata:
  name: grafana-service

spec:
  selector:
    app: grafana

  ports:
    - port: 3000
      targetPort: 3000

  type: LoadBalancer
```

Deploy:

```bash
kubectl apply -f grafana-deployment.yaml
```

---

# 🔗 Step 19 — Configure Grafana

Default Credentials:

```text
Username: admin
Password: admin
```

Add Prometheus datasource:

```text
http://prometheus-service:9090
```

---

# 🚀 Step 20 — Jenkinsfile

```groovy
pipeline {

    agent any

    environment {

        AWS_ACCOUNT_ID = "336984083625"
        AWS_REGION = "us-east-1"

        ECR_REPO = "devops-app"

        IMAGE_TAG = "latest"
    }

    stages {

        stage('Clone Repository') {

            steps {

                git 'https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git'
            }
        }

        stage('Build Docker Image') {

            steps {

                sh '''
                docker build -t $ECR_REPO:$IMAGE_TAG .
                '''
            }
        }

        stage('Login to ECR') {

            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'eks-aws-creds']
                ]) {

                    sh '''
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                    '''
                }
            }
        }

        stage('Tag Docker Image') {

            steps {

                sh '''
                docker tag $ECR_REPO:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Push Docker Image') {

            steps {

                sh '''
                docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Configure EKS Access') {

            steps {

                sh '''
                aws eks update-kubeconfig --region us-east-1 --name devops-cluster
                '''
            }
        }

        stage('Deploy to EKS') {

            steps {

                sh '''
                kubectl apply -f k8s-deployment.yaml
                kubectl apply -f hpa.yaml
                '''
            }
        }
    }

    post {

        success {
            echo 'Deployment Successful!'
        }

        failure {
            echo 'Deployment Failed!'
        }
    }
}
```

---

# ▶️ Step 21 — Create Jenkins Pipeline

Go to:

```text
Jenkins Dashboard → New Item
```

Choose:

```text
Pipeline
```

---

# ⚙️ Step 22 — Configure Pipeline

Select:

```text
Pipeline script from SCM
```

SCM:

```text
Git
```

Provide:

* Repository URL
* Branch → `main`
* Script Path → `Jenkinsfile`

Save Pipeline.

---

# ▶️ Step 23 — Run Pipeline

Click:

```text
Build Now
```

Pipeline executes:

✅ Git Clone
✅ Docker Build
✅ ECR Push
✅ EKS Deployment
✅ HPA Deployment

---

# 📈 Step 24 — Verify Deployment

```bash
kubectl get pods
kubectl get svc
kubectl get hpa
```

---

# 📊 Step 25 — Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify:

```bash
kubectl top nodes
kubectl top pods
```

---

# 🔥 Step 26 — Test Auto Scaling

Create load generator:

```bash
kubectl run -i --tty load-generator --image=busybox -- sh
```

Inside container:

```bash
while true; do wget -q -O- http://devops-service; done
```

Watch scaling:

```bash
kubectl get hpa -w
kubectl get pods -w
```

---

# 🚨 Troubleshooting

Refresh kubeconfig:

```bash
aws eks update-kubeconfig \
--region us-east-1 \
--name devops-cluster
```

---

# 💰 Cleanup

Delete cluster:

```bash
eksctl delete cluster \
--name devops-cluster \
--region us-east-1
```

---

# 🎯 Skills Demonstrated

✅ Jenkins
✅ Docker
✅ Kubernetes
✅ AWS EKS
✅ Amazon ECR
✅ CI/CD Pipelines
✅ Prometheus
✅ Grafana
✅ HPA
✅ Monitoring & Observability
✅ DevOps Automation
✅ Cloud-Native Deployment

---

# 🚀 Future Improvements

* Terraform Automation
* Helm Charts
* ArgoCD GitOps
* SonarQube Integration
* Trivy Security Scanning
* Slack Notifications
* Blue-Green Deployment

---

# 🙌 Final Outcome

Successfully implemented a complete production-style CI/CD pipeline using:

* Jenkins
* Docker
* AWS EKS
* Amazon ECR
* Kubernetes
* Prometheus
* Grafana
* HPA Auto Scaling

with fully automated cloud-native deployment workflows.
