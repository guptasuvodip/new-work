pipeline {
    agent any
    
    environment {
        // AWS Configuration - CHANGE THESE VALUES
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = sh(script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()
        
        // ECR Configuration - GET THIS FROM: terraform output ecr_repository_url
        ECR_REPOSITORY = 'my-website'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
        
        // EKS Configuration - GET THIS FROM: terraform output cluster_name
        EKS_CLUSTER_NAME = 'my-website-cluster'
        K8S_NAMESPACE = 'my-website'
        
        // Build Configuration
        IMAGE_TAG = "${BUILD_NUMBER}"
        GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '========== Checking out Code =========='
                checkout scm
                script {
                    env.GIT_BRANCH = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                    echo "Branch: ${env.GIT_BRANCH}"
                    echo "Commit: ${GIT_COMMIT_SHORT}"
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo '========== Running SonarQube Code Analysis =========='
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=my-website-project \
                                -Dsonar.projectName='My Website' \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions='**/terraform/**,**/k8s/**'
                        """
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                echo '========== Waiting for SonarQube Quality Gate =========='
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo '========== Building Docker Image =========='
                script {
                    dockerImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                    docker.build("${IMAGE_NAME}:latest")
                }
            }
        }
        
        stage('Trivy Security Scan') {
            steps {
                echo '========== Scanning Image for Vulnerabilities =========='
                script {
                    sh """
                        trivy image --severity HIGH,CRITICAL \
                            --exit-code 0 \
                            --format table \
                            ${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                echo '========== Pushing Image to Amazon ECR =========='
                script {
                    sh """
                        # Get AWS account ID
                        AWS_ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)
                        ECR_REGISTRY=\${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        IMAGE_NAME=\${ECR_REGISTRY}/${ECR_REPOSITORY}
                    
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        
                        echo "Image pushed: ${IMAGE_NAME}:${IMAGE_TAG}"
                    """
                }
            }
        }
        
        stage('Update Kubeconfig') {
            steps {
                echo '========== Configuring kubectl for EKS =========='
                sh """
                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER_NAME}
                    
                    kubectl version --client
                    kubectl cluster-info
                """
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                echo '========== Deploying to Kubernetes =========='
                script {
                    sh """
                        # Create namespace if doesn't exist
                        kubectl apply -f k8s/namespace.yaml
                        
                        # Update image in deployment
                        sed -i 's|IMAGE_URL_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|g' k8s/deployment.yaml
                        
                        # Apply manifests
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        
                        # Wait for rollout
                        kubectl rollout status deployment/website-deployment -n ${K8S_NAMESPACE} --timeout=5m
                        
                        echo "Deployment successful!"
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo '========== Verifying Deployment =========='
                script {
                    sh """
                        echo "Checking pods status..."
                        kubectl get pods -n ${K8S_NAMESPACE}
                        
                        echo ""
                        echo "Checking service..."
                        kubectl get service website-service -n ${K8S_NAMESPACE}
                        
                        echo ""
                        echo "Getting LoadBalancer URL..."
                        kubectl get service website-service -n ${K8S_NAMESPACE} \
                            -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo '=========================================='
            echo '✅ Pipeline completed successfully!'
            echo '=========================================='
            script {
                def lbUrl = sh(
                    script: "kubectl get service website-service -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                    returnStdout: true
                ).trim()
                
                echo "🌐 Access your website at: http://${lbUrl}"
            }
        }
        failure {
            echo '=========================================='
            echo '❌ Pipeline failed!'
            echo '=========================================='
        }
        always {
            echo 'Cleaning up...'
            sh 'docker system prune -f || true'
        }
    }
}
