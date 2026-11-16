pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        IMAGE_REPO = 'irontiger0000007/activity3healthcare'
        KUBE_NAMESPACE = 'activity3'
        DOCKERHUB_CREDENTIALS = 'dockerhub-creds'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'echo "Source code checked out successfully."'
                sh 'git rev-parse --short HEAD'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    set -eu

                    docker run --rm \
                        -v jenkins_home:/var/jenkins_home:ro \
                        -v trivy-cache:/root/.cache/ \
                        aquasec/trivy:0.76.2 \
                        fs \
                        --scanners vuln,misconfig,secret \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        "$WORKSPACE"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                        -t "$IMAGE_REPO:$BUILD_NUMBER" \
                        -f app/Dockerfile \
                        app
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    set -eu

                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v trivy-cache:/root/.cache/ \
                        aquasec/trivy:0.76.2 \
                        image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        "$IMAGE_REPO:$BUILD_NUMBER"
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USERNAME',
                        passwordVariable: 'DOCKERHUB_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x

                        echo "$DOCKERHUB_PASSWORD" |
                            docker login \
                                --username "$DOCKERHUB_USERNAME" \
                                --password-stdin

                        docker push "$IMAGE_REPO:$BUILD_NUMBER"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -eu

                    sed "s|irontiger0000007/activity3healthcare:dev|$IMAGE_REPO:$BUILD_NUMBER|g" \
                        k8s/deployment.yaml |
                        kubectl apply -f -

                    kubectl apply -f k8s/service.yaml

                    kubectl rollout status \
                        deployment/healthcare-web \
                        -n "$KUBE_NAMESPACE" \
                        --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "=== DEPLOYMENT ==="
                    kubectl get deployment healthcare-web -n "$KUBE_NAMESPACE"

                    echo "=== PODS ==="
                    kubectl get pods -n "$KUBE_NAMESPACE" -o wide

                    echo "=== SERVICE ==="
                    kubectl get service healthcare-service -n "$KUBE_NAMESPACE"

                    echo "=== SERVICE ENDPOINTS ==="
                    kubectl get endpoints healthcare-service -n "$KUBE_NAMESPACE"
                '''
            }
        }
    }

    post {
        success {
            echo 'Activity 3 DevSecOps pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the stage console output.'
        }
    }
}
