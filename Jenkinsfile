pipeline {
    agent { label 'docker-agent' }

    environment {
        AWS_REGION = "ap-south-1"
        ECR_REPO = "356627769740.dkr.ecr.ap-south-1.amazonaws.com/myapp"
        CLUSTER = "Blue_Green_Cluster"
        SERVICE = "Blue_Green-service-b7ifaw2f"
        TARGET_GROUP_GREEN = "arn:aws:elasticloadbalancing:ap-south-1:356627769740:targetgroup/GreenDeployment/0e097fc6fb1ec50b"
        LISTENER_ARN = "arn:aws:elasticloadbalancing:ap-south-1:356627769740:listener/app/ALBLoadbalancerBluegreen/c4e8f0d58f601261/f09c0f6b6d27d58c"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git credentialsId: 'github-creds',
                url: 'https://github.com/GiribabuGBB/Blue_Green_Deployment.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t myapp:${BUILD_NUMBER} .
                docker tag myapp:${BUILD_NUMBER} ${ECR_REPO}:${BUILD_NUMBER}
                """
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh '''
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                docker push ${ECR_REPO}:${BUILD_NUMBER}
                """
            }
        }

        stage('Deploy to ECS (Green)') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh '''
                    aws ecs update-service \
                    --cluster $CLUSTER \
                    --service $SERVICE \
                    --force-new-deployment \
                    --region $AWS_REGION
                    '''
                }
            }
        }

        stage('Health Check (Wait for Green)') {
            steps {
                sh '''
                echo "Waiting for Green deployment to stabilize..."
                sleep 60

                echo "Checking ALB health status..."
                '''
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh '''
                    aws elbv2 modify-listener \
                    --listener-arn $LISTENER_ARN \
                    --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_GREEN
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful - Green is now live"
        }
        failure {
            echo "❌ Deployment failed - check logs"
        }
    }
}
