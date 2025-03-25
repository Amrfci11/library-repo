pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "amrsayed11/library-app"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        EKS_CLUSTER = "my-eks-cluster"
        AWS_REGION = "eu-west-1"
        K8S_DIR = "k8s"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Amrfci11/library-repo.git',
                credentialsId: 'github-pip'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-creds') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                    }
                }
            }
        }

        stage('Configure AWS/EKS Access') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        sh """
                            # Configure kubectl access
                            aws eks update-kubeconfig \
                              --name ${EKS_CLUSTER} \
                              --region ${AWS_REGION} \
                              --kubeconfig ${KUBECONFIG} \
                              --alias eks-${EKS_CLUSTER}
                            
                            export KUBECONFIG=${KUBECONFIG}
                            
                            # Verify cluster access
                            kubectl config current-context
                            kubectl cluster-info
                        """
                    }
                }
            }
        }

        stage('Validate Network Connectivity') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        sh """
                            # Get EKS API endpoint
                            EKS_ENDPOINT=\$(aws eks describe-cluster \
                              --name ${EKS_CLUSTER} \
                              --region ${AWS_REGION} \
                              --query 'cluster.endpoint' \
                              --output text)
                            
                            # Test connectivity with timeout
                            if ! curl --max-time 15 -k -v \$EKS_ENDPOINT/api; then
                              echo "ERROR: Cannot reach EKS API endpoint"
                              exit 1
                            fi
                        """
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        retry(3) {
                            timeout(time: 5, unit: 'MINUTES') {
                                sh """
                                    export KUBECONFIG=${KUBECONFIG}
                                    
                                    # Apply Kubernetes manifests
                                    kubectl apply -f ${K8S_DIR}/namespace.yaml --validate=false
                                    kubectl apply -f ${K8S_DIR}/deployment.yaml
                                    kubectl apply -f ${K8S_DIR}/service.yaml
                                    
                                    # Verify deployment
                                    kubectl rollout status deployment/library-app \
                                      -n library-app \
                                      --timeout=300s
                                    
                                    # Get service details
                                    kubectl get svc -n library-app
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            sh "rm -f ${KUBECONFIG} || true"
        }
        success {
            echo "Successfully deployed ${DOCKER_IMAGE}:${DOCKER_TAG} to EKS"
        }
        failure {
            echo "Deployment failed - check network connectivity and EKS cluster status"
            sh """
                aws eks describe-cluster \
                  --name ${EKS_CLUSTER} \
                  --region ${AWS_REGION} \
                  --query 'cluster.{status:status,endpoint:endpoint}'
            """
        }
    }
}
