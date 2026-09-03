pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = '138893339858'
        AWS_REGION     = 'us-east-1'
        ECR_URL        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }
    stages {
        stage('ECR Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_URL'
                }
            }
        }
        stage('Build & Push Frontend') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    dir('frontend') {
                        sh 'docker build -t streaming-frontend .'
                        sh 'docker tag streaming-frontend:latest $ECR_URL/streaming-frontend:latest'
                        sh 'docker push $ECR_URL/streaming-frontend:latest'
                    }
                }
            }
        }
        stage('Build & Push Microservices') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    dir('backend') {
                        sh 'docker build -f authService/Dockerfile -t streaming-auth .'
                        sh 'docker tag streaming-auth:latest $ECR_URL/streaming-auth:latest'
                        sh 'docker push $ECR_URL/streaming-auth:latest'

                        sh 'docker build -f adminService/Dockerfile -t streaming-admin .'
                        sh 'docker tag streaming-admin:latest $ECR_URL/streaming-admin:latest'
                        sh 'docker push $ECR_URL/streaming-admin:latest'

                        sh 'docker build -f chatService/Dockerfile -t streaming-chat .'
                        sh 'docker tag streaming-chat:latest $ECR_URL/streaming-chat:latest'
                        sh 'docker push $ECR_URL/streaming-chat:latest'

                        sh 'docker build -f streamingService/Dockerfile -t streaming-service .'
                        sh 'docker tag streaming-service:latest $ECR_URL/streaming-service:latest'
                        sh 'docker push $ECR_URL/streaming-service:latest'
                    }
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'aws eks update-kubeconfig --region $AWS_REGION --name streaming-cluster-v3'
                    sh 'kubectl apply -f app-deployment.yaml'
                }
            }
        }
    }
}
