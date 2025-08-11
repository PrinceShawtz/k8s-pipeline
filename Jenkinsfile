pipeline {
    agent any
    
    environment {
        DOCKER_REPO = 'princeshawtz/k8s-pipeline'
    }
    
    parameters {
        string(name: 'K8S_NAMESPACE', defaultValue: 'default', description: 'Kubernetes namespace to deploy to')
    }
    
    stages {
        stage('Install kubectl') {
            steps {
                script {
                    // Check if kubectl exists, install if not
                    def kubectlExists = sh(script: 'which kubectl', returnStatus: true) == 0
                    if (!kubectlExists) {
                        echo "Installing kubectl..."
                        sh '''
                            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                            chmod +x kubectl
                            sudo mv kubectl /usr/local/bin/ || mv kubectl /usr/local/bin/kubectl
                            # Verify installation
                            kubectl version --client
                        '''
                    } else {
                        echo "kubectl already installed"
                        sh "kubectl version --client"
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    env.BUILD_ID = new Date().format('yyyyMMdd-HHmmss')
                    echo "Building Docker image: ${DOCKER_REPO}:${env.BUILD_ID}"
                    sh "docker build -t ${DOCKER_REPO}:${env.BUILD_ID} ."
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                echo "Pushing Docker image to Docker Hub"
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_REPO}:${BUILD_ID}
                        docker logout
                    '''
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "Deploying to Kubernetes namespace: ${params.K8S_NAMESPACE}"
                script {
                    // Check if deployment file exists in root directory
                    if (fileExists('deployment.yaml')) {
                        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                            sh '''
                                # Copy kubeconfig to a location we can use
                                mkdir -p ~/.kube
                                cp ${KUBECONFIG_FILE} ~/.kube/config
                                chmod 600 ~/.kube/config
                                
                                # Set KUBECONFIG environment variable
                                export KUBECONFIG=~/.kube/config
                                
                                # Verify kubeconfig format
                                echo "Checking kubeconfig format..."
                                if ! kubectl config view --minify > /dev/null 2>&1; then
                                    echo "❌ Invalid kubeconfig format"
                                    exit 1
                                fi
                                
                                # Test cluster connection with timeout
                                echo "Testing Kind cluster connection..."
                                if timeout 30s kubectl cluster-info > /dev/null 2>&1; then
                                    echo "✅ Kind cluster is accessible!"
                                    
                                    # Show current context
                                    kubectl config current-context
                                    
                                    # Create namespace if it doesn't exist
                                    kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                                    
                                    # Apply the deployment with correct variable substitution
                                    echo "Applying deployment to Kind cluster..."
                                    sed "s|__IMAGE__|${DOCKER_REPO}:${BUILD_ID}|g; s|__NAMESPACE__|${K8S_NAMESPACE}|g" deployment.yaml > deployment-final.yaml
                                    
                                    # Debug: show final deployment
                                    echo "Final deployment configuration:"
                                    cat deployment-final.yaml
                                    
                                    kubectl apply -f deployment-final.yaml
                                    
                                    # Wait for deployment to be ready
                                    echo "Waiting for deployment to be ready..."
                                    kubectl wait --for=condition=available --timeout=300s deployment/k8s-pipeline-deployment -n ${K8S_NAMESPACE}
                                    
                                    # Show deployment status
                                    echo "Deployment successful! Current status:"
                                    kubectl get deployments,pods,services -n ${K8S_NAMESPACE} -l app=k8s-pipeline
                                    
                                    # Get service info
                                    echo "Service details:"
                                    kubectl describe service k8s-pipeline-service -n ${K8S_NAMESPACE} || echo "Service not found"
                                    
                                    # Show how to access the application locally (Kind specific)
                                    echo ""
                                    echo "🚀 Application deployed successfully!"
                                    echo "📝 To access your app locally with Kind:"
                                    echo "   kubectl port-forward service/k8s-pipeline-service 8080:80 -n ${K8S_NAMESPACE}"
                                    echo "   Then visit: http://localhost:8080"
                                    
                                else
                                    echo "❌ Cannot connect to Kind cluster"
                                    echo "🔧 Troubleshooting steps:"
                                    echo "   1. Make sure Kind cluster is running: kind get clusters"
                                    echo "   2. Check kubeconfig: kubectl config view"
                                    echo "   3. Verify cluster status: docker ps | grep kind"
                                    echo "📦 Docker image was still built and pushed: ${DOCKER_REPO}:${BUILD_ID}"
                                    
                                    # Show detailed error information
                                    echo "Detailed cluster info:"
                                    kubectl cluster-info || true
                                    kubectl config get-contexts || true
                                    
                                    currentBuild.result = 'UNSTABLE'
                                fi
                            '''
                        }
                    } else {
                        error "Deployment file deployment.yaml not found in root directory!"
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "🎉 Pipeline completed successfully!"
            echo "🐳 Docker image: ${DOCKER_REPO}:${env.BUILD_ID}"
            echo "☸️  Deployed to Kind cluster in namespace: ${params.K8S_NAMESPACE}"
            echo "🔗 Docker Hub: https://hub.docker.com/r/${DOCKER_REPO}/tags"
            echo ""
            echo "💡 Next steps:"
            echo "   • Test your app: kubectl port-forward service/k8s-pipeline-service 8080:80 -n ${params.K8S_NAMESPACE}"
            echo "   • View logs: kubectl logs -l app=k8s-pipeline -n ${params.K8S_NAMESPACE}"
            echo "   • Scale up: kubectl scale deployment k8s-pipeline-deployment --replicas=3 -n ${params.K8S_NAMESPACE}"
            
            // Clean up local Docker image to save space
            sh "docker rmi ${DOCKER_REPO}:${env.BUILD_ID} || true"
        }
        failure {
            echo "❌ Pipeline failed."
            sh "docker rmi ${DOCKER_REPO}:${env.BUILD_ID} || true"
        }
        always {
            sh "docker system prune -f || true"
        }
    }
}
