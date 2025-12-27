pipeline {
    agent any

    environment {
        DOCKER_HUB_REPO = 'sai2001docker/k8s-ingress-prj-app'
        K8S_CLUSTER_NAME = 'my-cluster'
        AWS_REGION = 'ap-south-1'
        NAMESPACE = 'default'
        APP_NAME = 'techsolutions'
        IMAGE_TAG = 'latest'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/UppalavenkataSai/k8s-ingress-project-.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = env.BUILD_NUMBER
                }
                sh """
                    docker build -t ${DOCKER_HUB_REPO}:${IMAGE_TAG} .
                    docker tag ${DOCKER_HUB_REPO}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:latest
                """
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh """
                        echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                        docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                        docker push ${DOCKER_HUB_REPO}:latest
                    """
                }
            }
        }

        stage('Configure AWS and Kubectl') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh """
                        aws configure set region ${AWS_REGION}
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${K8S_CLUSTER_NAME}
                        kubectl get nodes
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh """
                        sed -i "s|${DOCKER_HUB_REPO}:latest|${DOCKER_HUB_REPO}:${IMAGE_TAG}|g" k8s/deployment.yaml
                        kubectl apply -f k8s/deployment.yaml --validate=false
                        kubectl rollout status deployment/${APP_NAME}-deployment --timeout=300s
                    """
                }
            }
        }

        stage('Deploy Ingress') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh """
                        kubectl apply -f k8s/ingress.yaml
                        kubectl get ingress
                    """
                }
            }
        }
    }

    post {
        always {
            sh """
                docker rmi ${DOCKER_HUB_REPO}:${IMAGE_TAG} || true
                docker rmi ${DOCKER_HUB_REPO}:latest || true
            """
        }

        success {
            echo "✅ Pipeline completed successfully"
        }

        failure {
            echo "❌ Pipeline failed"
        }
    }
}
