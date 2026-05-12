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
                git branch: 'main',
                url: 'https://github.com/govindsainidevops/ci-cd-pipeline-k8s-eks-Jenkins.git'
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