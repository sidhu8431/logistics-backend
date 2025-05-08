pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        DOCKER_IMAGE_NAME = "sarusparks/logistics_be"
        IMAGE_TAG = "${BUILD_NUMBER}"
        KUBECONFIG_PATH = "/tmp/kubeconfig"
        HELM_RELEASE_NAME = "logistics-backend"
        HELM_CHART_DIR = "Helm/backend"
        AWS_REGION = "us-east-2"
        K8S_NAMESPACE = "logistics"
    }

    stages {
        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Checking Docker..."
                    if ! [ -x "$(command -v docker)" ]; then
                        echo "Installing Docker..."
                        sudo yum update -y
                        sudo yum install -y docker
                        sudo systemctl start docker
                        sudo systemctl enable docker
                        sudo usermod -aG docker $(whoami)
                    fi

                    echo "Checking Helm..."
                    if ! [ -x "$(command -v helm)" ]; then
                        curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash
                    fi

                    echo "Checking AWS CLI..."
                    if ! [ -x "$(command -v aws)" ]; then
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip awscliv2.zip
                        sudo ./aws/install
                    fi

                    echo "Checking kubectl..."
                    if ! [ -x "$(command -v kubectl)" ]; then
                        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl && sudo mv kubectl /usr/local/bin/
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

        stage('Build Application') {
            steps {
                script {
                    if (fileExists('pom.xml')) {
                        sh 'mvn clean install -Dmaven.test.skip=true'
                    } else {
                        error "pom.xml not found!"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-access', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    dir('Docker/backend/') {
                        sh '''
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                            docker build -t ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} .
                            docker push ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }

        stage('Configure Kubeconfig') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'sidhu_aws_access']]) {
                    sh '''
                        aws eks update-kubeconfig --name logistics-cluster --region ${AWS_REGION}
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('Ensure Namespace Exists') {
            steps {
                sh '''
                    if ! kubectl get namespace ${K8S_NAMESPACE}; then
                        kubectl create namespace ${K8S_NAMESPACE}
                    fi
                '''
            }
        }

        stage('Helm Deploy') {
            steps {
                dir("${HELM_CHART_DIR}") {
                    sh '''
                        helm upgrade --install ${HELM_RELEASE_NAME} . \
                          --set image.repository=${DOCKER_IMAGE_NAME} \
                          --set image.tag=${IMAGE_TAG} \
                          --set service.type=LoadBalancer \
                          --set service.port=8080 \
                          --set service.targetPort=8080 \
                          --namespace ${K8S_NAMESPACE} --create-namespace
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
