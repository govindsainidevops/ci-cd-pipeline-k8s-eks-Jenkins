# ci-cd-pipeline-k8s-eks-Jenkins

# 🚀 End-to-End CI/CD Pipeline with Jenkins + AWS EKS

## 📌 Project Overview

This project demonstrates a complete production-style DevOps workflow using:

* Jenkins
* Docker
* Amazon ECR
* Kubernetes
* AWS EKS
* Prometheus
* Grafana
* Horizontal Pod Autoscaler (HPA)

The pipeline automates:

1. Source Code Checkout
2. Docker Image Build
3. Docker Image Push to ECR
4. Kubernetes Deployment to AWS EKS
5. Monitoring Setup
6. Auto Scaling

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

Install the following tools:

| Tool             | Purpose            |
| ---------------- | ------------------ |
| AWS CLI          | AWS Authentication |
| kubectl          | Kubernetes CLI     |
| eksctl           | Create EKS Cluster |
| Docker Desktop   | Docker Runtime     |
| Jenkins (Docker) | CI/CD Automation   |

---

# 🔐 Step 1 — Configure AWS CLI

```bash
aws configure
```

Provide:

* AWS Access Key
* AWS Secret Access Key
* Region → `us-east-1`
* Output Format → `json`

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

Expected Output:

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

# 🖥️ Step 5 — Jenkins Setup (Docker Desktop)

Jenkins is running locally using Docker Desktop.

Access Jenkins:

```text
http://localhost:8080
```

---

# 🚀 Step 6 — Run Jenkins Container

Run Jenkins with Docker socket access:

```bash
docker run -d \
--name jenkins \
-p 8080:8080 \
-p 50000:50000 \
-v jenkins_home:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock \
jenkins/jenkins:lts
```

---

# 🔓 Step 7 — Unlock Jenkins

Get initial Jenkins password:

```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Install suggested plugins.

Create admin user.

---

# 🔌 Step 8 — Install Jenkins Plugins

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
* Blue Ocean

Restart Jenkins container if needed.

---

# 🔑 Step 9 — Configure Jenkins Credentials

Go to:

```text
Manage Jenkins → Credentials
```

Add AWS Credentials:

| Field      | Value               |
| ---------- | ------------------- |
| Kind       | AWS Credentials     |
| ID         | aws-creds           |
| Access Key | Your AWS Access Key |
| Secret Key | Your AWS Secret Key |

---

# 🐳 Step 10 — Enable Docker Access

In Docker Desktop:

```text
Settings → General
```

Enable:

```text
Expose daemon on tcp://localhost:2375 without TLS
```

Restart Docker Desktop.

---

# ☸️ Step 11 — Configure kubectl Access Inside Jenkins

Copy kubeconfig into Jenkins container:

```bash
docker cp ~/.kube jenkins:/var/jenkins_home/.kube
```

Access Jenkins container:

```bash
docker exec -it jenkins bash
```

Verify Kubernetes access:

```bash
kubectl get nodes
```

You should see EKS nodes in `Ready` state.

---

# 📂 Step 12 — Project Structure

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

# 🐳 Step 13 — Dockerfile

Create:

```text
Dockerfile
```

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

# ☸️ Step 14 — Kubernetes Deployment File

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

# 🚀 Step 15 — Create Jenkinsfile

Create:

```text
Jenkinsfile
```

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
                    credentialsId: 'aws-creds']
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

        stage('Deploy to EKS') {

            steps {

                sh '''
                kubectl apply -f k8s-deployment.yaml
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

# 🚀 Step 16 — Create Jenkins Pipeline Job

Go to:

```text
Jenkins Dashboard → New Item
```

Choose:

* Pipeline

---

# ⚙️ Step 17 — Configure Pipeline

## Pipeline Source

```text
Pipeline script from SCM
```

SCM:

* Git

Repository URL:

* Your GitHub Repository URL

Branch:

```text
main
```

Script Path:

```text
Jenkinsfile
```

Save the job.

---

# ▶️ Step 18 — Run Jenkins Pipeline

Click:

```text
Build Now
```

Pipeline stages:

* Clone Repository
* Build Docker Image
* Login to ECR
* Push Image
* Deploy to EKS

---

# 🔍 Step 19 — Verify Deployment

```bash
kubectl get pods
kubectl get svc
```

---

# 🌍 Step 20 — Access Application

```bash
kubectl get svc
```

Open:

```text
http://<EXTERNAL-IP>
```

---

# 📊 Step 21 — Deploy Prometheus

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

Apply:

```bash
kubectl apply -f prometheus-deployment.yaml
```

---

# 📈 Step 22 — Deploy Grafana

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

Apply:

```bash
kubectl apply -f grafana-deployment.yaml
```

---

# 🔗 Step 23 — Configure Grafana

Default Credentials:

```text
Username: admin
Password: admin
```

Prometheus URL:

```text
http://prometheus-service:9090
```

---

# ⚖️ Step 24 — Configure HPA

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

---

# 📈 Step 25 — Verify HPA

```bash
kubectl get hpa
```

---

# 📊 Step 26 — Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify:

```bash
kubectl top nodes
kubectl top pods
```

---

# 🔥 Step 27 — Generate Load for HPA Testing

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

# 🛠️ Useful Commands

## Get Pods

```bash
kubectl get pods
```

## Get Services

```bash
kubectl get svc
```

## View Logs

```bash
kubectl logs deployment/devops-app
```

## Restart Deployment

```bash
kubectl rollout restart deployment devops-app
```

---

# 🚨 Troubleshooting

## Update kubeconfig

```bash
aws eks update-kubeconfig \
--region us-east-1 \
--name devops-cluster
```

---

## Verify Context

```bash
kubectl config get-contexts
```

---

## Verify ECR Images

```bash
aws ecr describe-images \
--repository-name devops-app \
--region us-east-1
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

# 🚀 Future Improvements

* Terraform Automation
* Helm Charts
* ArgoCD GitOps
* SonarQube Integration
* Trivy Security Scanning
* Slack Notifications
* Blue-Green Deployment
* Canary Deployment

---

# 🎯 Resume-Worthy Skills Demonstrated

✅ Jenkins
✅ Docker
✅ Kubernetes
✅ AWS EKS
✅ Amazon ECR
✅ CI/CD Pipelines
✅ Prometheus
✅ Grafana
✅ HPA
✅ Cloud Deployment
✅ Monitoring & Observability
✅ DevOps Automation

---

# 🙌 Final Outcome

Successfully implemented a production-style CI/CD pipeline using Jenkins and AWS EKS with:

* Automated Docker builds
* ECR image push
* Kubernetes deployment
* Monitoring stack
* Auto scaling
* Cloud-native deployment workflow
