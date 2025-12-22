pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKER_IMAGE = 'louay1732001/petclinic'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/LouayChayeb/petclinic.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                    docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh """
                    echo \$DOCKER_HUB_CREDENTIALS_PSW | docker login -u \$DOCKER_HUB_CREDENTIALS_USR --password-stdin
                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE}:latest
                    docker logout
                """
            }
        }

        stage('Prepare Minikube') {
            steps {
                sh """
                    echo "=== Checking Minikube Status ==="
                    minikube status || minikube start --driver=docker

                    echo "=== Pre-pulling Docker Image in Minikube ==="
                    minikube ssh "docker pull ${DOCKER_IMAGE}:latest"
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "=== Deploying to Kubernetes ==="
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    echo "=== Waiting for Rollout ==="
                    kubectl rollout status deployment/springpetclinic --timeout=5m

                    echo "=== Deployment Info ==="
                    kubectl get pods -l app=springpetclinic
                    kubectl get svc springpetclinic

                    echo "=== Service URL ==="
                    minikube service springpetclinic --url
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful!"
            echo "Access the application: minikube service springpetclinic"
        }
        failure {
            echo "❌ Deployment failed!"
            emailext (
                subject: "Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                <h2>Build Failed</h2>
                <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                <p><strong>Console Output:</strong> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                <br>
                <p>Please check the console output for details.</p>
                """,
                to: 'louaychayeb00@gmail.com',
                mimeType: 'text/html'
            )
        }
    }
}
