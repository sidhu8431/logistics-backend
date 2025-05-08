pipeline {
    agent any

    tools {
        maven 'maven' // Name from Global Tool Configuration
    }

    environment {
        DOCKER_IMAGE_NAME = "sarusparks/logistics_be"
        KUBECONFIG_PATH = "/tmp/kubeconfig"
        HELM_RELEASE_NAME = "logistics-backend"
        HELM_CHART_DIR = "Helm/backend"
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

        stage('Checkout Code from GitHub') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sidhu8431/logistics-backend.git'
            }
        }

        stage('Build Application') {
            steps {
              
                    script {
                        if (fileExists('pom.xml')) {
                            sh 'mvn clean install -DskipTests'
                        } else {
                            error "pom.xml not found in the logistics-backend directory!"
                        }
                    }
                
            }
        }

        stage('Docker Build and Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-access', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    dir('Docker/backend/') {
                        sh '''
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                            docker build -t ${DOCKER_IMAGE_NAME}:v1 .
                            docker push ${DOCKER_IMAGE_NAME}:v1
                        '''
                    }
                }
            }
        }

        stage('Configure kubeconfig for EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'sidhu_aws_access']]) {
                    sh '''
                        echo "Generating kubeconfig for EKS..."
                        aws eks update-kubeconfig --name logistics-cluster --region ${AWS_REGION}
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('Fix Helm Namespace Labels') {
            steps {
                sh '''
                    echo "Adding Helm metadata to namespace if missing..."
                    helm install ${HELM_RELEASE_NAME} Helm/backend/ --namespace logistics --create-namespace
                '''
            }
        }

        stage('Helm Deploy') {
            steps {
                dir('Helm/backend/') {
                    sh '''
                        echo "Deploying with Helm..."
                        helm upgrade --install ${HELM_RELEASE_NAME} ${HELM_CHART_DIR} \
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
