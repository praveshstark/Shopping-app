pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        ECR_REPO = 'shopping-app'
        IMAGE_TAG = "shoppingapp"
        AWS_ACCOUNT_ID = '424522917744'
        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build JAR') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
    steps {
        script {
            sh """
                docker buildx build \
                --platform linux/amd64 \
                -t ${ECR_REPO}:${IMAGE_TAG} \
                --load .
            """

            sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}"
        }
    }
}

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    """
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh "docker push ${ECR_URI}:${IMAGE_TAG}"
            }
        }
    }

    stage('Deploy to ECS') {
    steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
            sh """
                # Register new task definition revision
                aws ecs register-task-definition \
                    --family shoppingapp-task \
                    --network-mode awsvpc \
                    --requires-compatibilities FARGATE \
                    --cpu 256 \
                    --memory 512 \
                    --execution-role-arn arn:aws:iam::424522917744:role/ecsTaskExecutionRole \
                    --container-definitions '[
                        {
                            "name": "shoppingapp",
                            "image": "${ECR_URI}:${IMAGE_TAG}",
                            "essential": true,
                            "portMappings": [
                                {
                                    "containerPort": 8080,
                                    "protocol": "tcp"
                                }
                            ]
                        }
                    ]' \
                    --region ap-southeast-2

                # Update service with new revision
                aws ecs update-service \
                    --cluster shoppingapp \
                    --service shoppingapp-service \
                    --force-new-deployment \
                    --region ap-southeast-2
            """
        }
    }
}

    post {
        success {
            echo '🚀 Deployment Successful!'
        }
        failure {
            echo '❌ Deployment Failed!'
        }
    }
}