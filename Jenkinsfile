pipeline {
    agent any
    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REGISTRY = '503577850130.dkr.ecr.ap-south-1.amazonaws.com'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/sowjanya-gundumalla/StreamingApp.git'
            }
        }
        stage('Login to ECR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin $ECR_REGISTRY'
                }
            }
        }
        stage('Build and Push Frontend') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'docker build -t streaming-frontend ./frontend'
                    sh 'docker tag streaming-frontend:latest $ECR_REGISTRY/streaming-frontend:latest'
                    sh 'docker push $ECR_REGISTRY/streaming-frontend:latest'
                }
            }
        }
        stage('Build and Push Auth') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'docker build -t streaming-auth ./backend/authService'
                    sh 'docker tag streaming-auth:latest $ECR_REGISTRY/streaming-auth:latest'
                    sh 'docker push $ECR_REGISTRY/streaming-auth:latest'
                }
            }
        }
        stage('Build and Push Streaming') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'docker build -t streaming-service -f ./backend/streamingService/Dockerfile ./backend'
                    sh 'docker tag streaming-service:latest $ECR_REGISTRY/streaming-service:latest'
                    sh 'docker push $ECR_REGISTRY/streaming-service:latest'
                }
            }
        }
        stage('Build and Push Admin') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'docker build -t streaming-admin -f ./backend/adminService/Dockerfile ./backend'
                    sh 'docker tag streaming-admin:latest $ECR_REGISTRY/streaming-admin:latest'
                    sh 'docker push $ECR_REGISTRY/streaming-admin:latest'
                }
            }
        }
        stage('Build and Push Chat') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'docker build -t streaming-chat -f ./backend/chatService/Dockerfile ./backend'
                    sh 'docker tag streaming-chat:latest $ECR_REGISTRY/streaming-chat:latest'
                    sh 'docker push $ECR_REGISTRY/streaming-chat:latest'
                }
            }
        }
    }
    post {
        success {
            echo 'All images built and pushed to ECR successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
