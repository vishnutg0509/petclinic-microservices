pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_DEFAULT_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '632699415921'
        ECR_REGISTRY   = '632699415921.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_PREFIX     = 'petclinic-dev'
        KUBECONFIG     = '/var/lib/jenkins/.kube/config'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git branch --show-current'
            }
        }

        stage('Maven Build and Test') {
            steps {
                sh '''
                    set -eu
                    chmod +x ./mvnw
                    ./mvnw --no-transfer-progress --batch-mode clean verify
                '''
            }
        }

        stage('Prepare Image Tag') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Docker image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build and Push Images to ECR') {
            steps {
                sh '''
                    set -eu

                    export DOCKER_CONFIG="$WORKSPACE/.docker-ecr"
                    mkdir -p "$DOCKER_CONFIG"

                    cleanup() {
                        docker logout "$ECR_REGISTRY" >/dev/null 2>&1 || true
                        rm -rf "$DOCKER_CONFIG"
                    }
                    trap cleanup EXIT

                    aws ecr get-login-password --region "$AWS_REGION" |
                        docker login --username AWS --password-stdin "$ECR_REGISTRY"

                    for entry in \
                        "config-server:8888" \
                        "discovery-server:8761" \
                        "customers-service:8081" \
                        "visits-service:8082" \
                        "vets-service:8083" \
                        "admin-server:9090" \
                        "api-gateway:8080"
                    do
                        service="${entry%%:*}"
                        port="${entry##*:}"
                        artifact="spring-petclinic-${service}"
                        image="${ECR_REGISTRY}/${ECR_PREFIX}/${service}"

                        echo "Building ${service}"

                        cp "${artifact}/target/${artifact}-"*.jar "${artifact}.jar"

                        docker buildx build \
                            --platform linux/amd64 \
                            --build-arg ARTIFACT_NAME="$artifact" \
                            --build-arg EXPOSED_PORT="$port" \
                            -f docker/Dockerfile \
                            -t "${image}:${IMAGE_TAG}" \
                            --push \
                            .
                    done
                '''
            }
        }

        stage('Deploy to EKS with Helm') {
            steps {
                sh '''
                    set -eu

                    kubectl get nodes

                    for service in \
                        config-server \
                        discovery-server \
                        customers-service \
                        visits-service \
                        vets-service \
                        admin-server \
                        api-gateway
                    do
                        echo "Deploying ${service}"

                        helm upgrade --install "$service" \
                            ./helm/petclinic-service \
                            --namespace petclinic \
                            --values "helm-values/${service}.yaml" \
                            --set-string image.tag="$IMAGE_TAG" \
                            --wait \
                            --timeout 5m

                        kubectl rollout status deployment/"$service" -n petclinic --timeout=180s
                    done
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -eu

                    echo "=== Helm releases ==="
                    helm list --namespace petclinic

                    echo "=== Kubernetes deployments ==="
                    kubectl get deployments -n petclinic

                    echo "=== Kubernetes pods ==="
                    kubectl get pods -n petclinic -o wide

                    echo "=== Kubernetes services ==="
                    kubectl get services -n petclinic
                '''
            }
        }
    }

    post {
        success {
            echo 'PetClinic CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the failed stage in Jenkins console output.'
        }
    }
}
