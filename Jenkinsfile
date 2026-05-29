pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        TF_VAR_key_name       = credentials('aws-key-pair-name')
        IMAGE_NAME            = 'hello-world'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Plan') {
            steps {
                dir('terraform') {
                    sh 'terraform init -input=false'
                    sh 'terraform plan -input=false -out=tfplan'
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                dir('terraform') {
                    sh 'terraform apply -auto-approve tfplan'
                    script {
                        env.EC2_IP = sh(
                            script: 'terraform output -raw instance_public_ip',
                            returnStdout: true
                        ).trim()
                    }
                }
                echo "Instance IP: ${env.EC2_IP}"
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:latest ./app"
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 ubuntu@${env.EC2_IP} 'echo ready'; do
                            echo 'Waiting for instance...'
                            sleep 10
                        done

                        ssh -o StrictHostKeyChecking=no ubuntu@${env.EC2_IP} '
                            if ! command -v docker &> /dev/null; then
                                sudo apt-get update -y
                                sudo apt-get install -y docker.io
                                sudo systemctl enable --now docker
                            fi
                        '

                        docker save ${IMAGE_NAME}:latest | ssh -o StrictHostKeyChecking=no ubuntu@${env.EC2_IP} 'sudo docker load'

                        ssh -o StrictHostKeyChecking=no ubuntu@${env.EC2_IP} '
                            sudo docker stop ${IMAGE_NAME} 2>/dev/null || true
                            sudo docker rm   ${IMAGE_NAME} 2>/dev/null || true
                            sudo docker run -d --name ${IMAGE_NAME} --restart always -p 80:3000 ${IMAGE_NAME}:latest
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deploy completato: http://${env.EC2_IP}"
        }
        failure {
            echo "Pipeline fallita"
        }
    }
}
