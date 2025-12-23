pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKER_IMAGE = 'louay1732001/petclinic'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
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
            steps { sh ''' echo "Skipping Docker build - using existing image"
            # docker logout # echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
            # docker build -t $DOCKER_USERNAME/petclinic:latest .
            # docker push $DOCKER_USERNAME/petclinic:latest ''' }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "=== Deleting old deployment ==="
                    kubectl delete deployment springpetclinic --ignore-not-found=true
                    kubectl delete service springpetclinic --ignore-not-found=true

                    echo "=== Deploying to Kubernetes ==="
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    echo "=== Waiting for Rollout ==="
                    kubectl rollout status deployment/springpetclinic --timeout=10m

                    echo "=== Deployment Info ==="
                    kubectl get pods -l app=springpetclinic
                    kubectl get svc springpetclinic

                    echo "=== Getting Minikube IP ==="
                    MINIKUBE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
                    echo "Application accessible at: http://$MINIKUBE_IP:30080"
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful!"
            sh '''
                MINIKUBE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
                echo "================================================"
                echo "Application URL: http://$MINIKUBE_IP:30080"
                echo "================================================"
            '''
             emailext (
                            subject: "Jenkins Build success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                            body: """
                            <h2>Build SUCCESS</h2>
                            <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                            <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                            <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                            <p><strong>Console Output:</strong> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                            """,
                            to: 'louaychayeb00@gmail.com',
                            mimeType: 'text/html'
                        )
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
                """,
                to: 'louaychayeb00@gmail.com',
                mimeType: 'text/html'
            )
        }
        always {
            sh '''
                echo "=== Cleaning up old Docker images ==="
                docker image prune -f
            '''
        }
    }
}
