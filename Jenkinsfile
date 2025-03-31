pipeline {
    agent any

    environment{
        AWS_REGION = 'ap-northeast-1'
        IMAGE_NAME = 'cryptory-gateway'
        ECR_REGISTRY = '050314037804.dkr.ecr.ap-northeast-1.amazonaws.com'
        ECR_REPO = "${ECR_REGISTRY}/${IMAGE_NAME}"
    }

    stages {
        stage('Checkout') {  // 1️⃣ GitHub에서 코드 가져오기
            steps {
                git branch: 'main', url: 'https://github.com/lgcns-2nd-project-Ix4/Cryptory-Gateway.git'
            }
        }

        stage('Build Docker Image') {
			steps {
				sh """
                    docker build -t $IMAGE_NAME .
                    docker tag $IMAGE_NAME:latest $ECR_REPO:latest
                    echo docker build success
                """
            }
        }

        stage('Login to ECR') {
			steps {
				withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'AWS-CREDENTIALS'
                ]]) {
					sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY'
                }
            }
        }

        stage('Push Docker Image') {
			steps {
				sh 'docker push $ECR_REPO:latest'
            }
        }

        stage('Deploy to EC2'){
			steps{
				withEnv([
                    "AWS_REGION=${env.AWS_REGION}",
                    "ECR_REPO=${env.ECR_REPO}",
                    "ECR_REGISTRY=${env.ECR_REGISTRY}",
                    "GIT_USERNAME=${GIT_USERNAME}",
                    "GIT_PASSWORD=${GIT_PASSWORD}",
                    "CONFIG_SERVER_URL=${CONFIG_SERVER_URL}",
                    "IMAGE_NAME=${env.IMAGE_NAME}",
                    "RABBITMQ_HOST=${env.RABBITMQ_HOST}",
                    "RABBITMQ_PORT=${env.RABBITMQ_PORT}",
                    "RABBITMQ_USERNAME=${env.RABBITMQ_USERNAME}",
                    "RABBITMQ_PASSWORD=${env.RABBITMQ_PASSWORD}",
                    "EUREKKA_URL=${env.EUREKA_URL}"
                ]){
					sshPublisher(
                        publishers: [
                            sshPublisherDesc(
                                configName: 'Outer-Server',
                                transfers: [
                                    sshTransfer(
                                        cleanRemote: false,
                                        excludes: '',
                                        execCommand: """
                                            set -x

                                            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                                            docker pull $ECR_REPO:latest
                                            docker stop $IMAGE_NAME || true
                                            docker rm $IMAGE_NAME || true
                                            docker run -d --name $IMAGE_NAME -p 8080:8080 \\
                                                -e CONFIG_SERVER_URL=$CONFIG_SERVER_URL \\
                                                -e RABBITMQ_HOST=$RABBITMQ_HOST \\
                                                -e RABBITMQ_PORT=$RABBITMQ_PORT \\
                                                -e RABBITMQ_USERNAME=$RABBITMQ_USERNAME \\
                                                -e RABBITMQ_PASSWORD=$RABBITMQ_PASSWORD \\
                                                -e SPRING_PROFILE_ACTIVE=docker \\
                                                -e EUREKA_URL=$EUREKA_URL \\
                                                $ECR_REPO:latest
                                            echo "Deploy successful"
                                        """,
                                        execTimeout: 120000,
                                        flatten: false,
                                        )
                                ],
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: false) // verbose옵션으로 쉘 출력이 표시됨.
                        ]
                    )
                }
            }
        }
    }


   post {
		success {
			echo "✅ Docker image successfully pushed to ECR!"
        }
        failure {
			echo "❌ Docker image push failed!"
        }
    }
}
