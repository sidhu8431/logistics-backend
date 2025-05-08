pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        DOCKER_IMAGE_NAME = "sarusparks/logistics_be"
        HELM_RELEASE_NAME = "logistics-backend"
        AWS_REGION = "us-east-2"
    }

    stages {
        stage('Install Docker and Helm') {
            steps {
                sh '''
                    if ! [ -x "$(command -v docker)" ]; then
                        echo "Installing Docker..."
                        sudo yum update -y
                        sudo yum install -y docker
                        sudo systemctl start docker
                        sudo systemctl enable docker
                        sudo usermod -aG docker $(whoami)
                    else
                        echo "Docker is already installed."
                    fi

                    if ! [ -x "$(command -v helm)" ]; then
                        echo "Installing Helm..."
                        curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash
                    else
                        echo "Helm is already installed."
                    fi
                '''
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sidhu8431/logistics-backend.git'
            }
        }

        stage('Build with Maven') {
            steps {
                dir('logistics-backend') {
                    sh 'mvn clean install -DskipTests'
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-access', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    dir('Docker/backend') {
                        sh '''
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                            docker build -t ${DOCKER_IMAGE_NAME}:v1 .
                            docker push ${DOCKER_IMAGE_NAME}:v1
                        '''
                    }
                }
            }
        }

        stage('Configure kubeconfig') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'sidhu_aws_access']]) {
                    sh '''
                        aws eks update-kubeconfig --name logistics-cluster --region ${AWS_REGION}
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('Fix Helm Namespace Labels') {
            steps {
                dir('Helm/backend') {
                    sh '''
                        kubectl label namespace logistics app.kubernetes.io/managed-by=Helm --overwrite
                        kubectl annotate namespace logistics \
                          meta.helm.sh/release-name=${HELM_RELEASE_NAME} \
                          meta.helm.sh/release-namespace=logistics --overwrite
                    '''
                }
            }
        }

        stage('Helm Deploy') {
            steps {
                dir('Helm/backend') {
                    sh '''
                        helm upgrade --install ${HELM_RELEASE_NAME} . \
                          --set image.repository=${DOCKER_IMAGE_NAME} \
                          --set image.tag=v1 \
                          --set service.type=LoadBalancer \
                          --set service.port=8080 \
                          --set service.targetPort=8080 \
                          --namespace logistics --create-namespace
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "\u001B[32m✔ Pipeline completed successfully.\u001B[0m"
        }
        failure {
            echo "\u001B[31m✘ Pipeline failed. Please check the logs.\u001B[0m"
        }
    }
}
