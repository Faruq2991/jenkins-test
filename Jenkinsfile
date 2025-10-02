pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'faruq96/jenkins-app'  // Change to your DockerHub username
        APP_VERSION = "${env.BUILD_NUMBER}"
        K8S_NAMESPACE = 'jenkins-test'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '📥 Pulling code from GitHub...'
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo '📦 Installing dependencies...'
                sh 'npm install'
            }
        }
        
        stage('Run Tests') {
            steps {
                echo '🧪 Running tests...'
                sh 'npm test'
            }
            post {
                always {
                    // Publish test results if available
                    junit allowEmptyResults: true, testResults: '**/junit.xml'
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image: ${DOCKER_IMAGE}:${APP_VERSION}"
                sh ''' 
                export DOCKER_HOST=$(minikube docker-env --shell bash | grep DOCKER_HOST | cut -d= -f2) # This makes Jenkins use Minikube’s Docker daemon.
                docker build -t ${DOCKER_IMAGE}:${APP_VERSION} .
                docker tag ${DOCKER_IMAGE}:${APP_VERSION} ${DOCKER_IMAGE}:latest
                '''
            }
        }
        
        stage('Push to DockerHub') {
            steps {
                echo '📤 Pushing image to DockerHub...'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        sh """
                            docker push ${DOCKER_IMAGE}:${APP_VERSION}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        
        stage('Deploy to Kubernetes') {
            steps {
                echo '🚀 Deploying to local Kubernetes cluster...'
                sh """
                    # Update image version in deployment
                    sed 's|BUILD_NUMBER_PLACEHOLDER|${APP_VERSION}|g' k8s-deployment.yaml | kubectl apply -f - -n ${K8S_NAMESPACE}
                    
                    # Set the new image
                    kubectl set image deployment/myapp-deployment \
                        myapp=${DOCKER_IMAGE}:${APP_VERSION} \
                        -n ${K8S_NAMESPACE}
                    
                    # Wait for rollout
                    kubectl rollout status deployment/myapp-deployment \
                        -n ${K8S_NAMESPACE} --timeout=3m
                """
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo '✅ Verifying deployment...'
                sh """
                    echo "Pods:"
                    kubectl get pods -n ${K8S_NAMESPACE}
                    
                    echo "\\nServices:"
                    kubectl get svc -n ${K8S_NAMESPACE}
                    
                    echo "\\nDeployment:"
                    kubectl get deployment -n ${K8S_NAMESPACE}
                """
            }
        }
        
        stage('Smoke Test') {
            steps {
                echo '🔍 Running smoke tests...'
                script {
                    sh """
                        # Wait a bit for service to be ready
                        sleep 10
                        
                        # Get Minikube service URL
                        SERVICE_URL=\$(minikube service myapp-service -n ${K8S_NAMESPACE} --url)
                        
                        echo "Testing endpoint: \$SERVICE_URL/health"
                        
                        # Test health endpoint
                        HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" \$SERVICE_URL/health)
                        
                        if [ \$HTTP_CODE -eq 200 ]; then
                            echo "✅ Health check passed (HTTP \$HTTP_CODE)"
                        else
                            echo "❌ Health check failed (HTTP \$HTTP_CODE)"
                            exit 1
                        fi
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline completed successfully!'
            echo "Access your app: minikube service myapp-service -n ${K8S_NAMESPACE}"
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        always {
            echo '🧹 Cleaning up...'
            sh 'rm -rf node_modules || true'
        }
    }
}