pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "amrsayed11/library-app"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        EKS_CLUSTER = "my-eks-cluster"
        AWS_REGION = "eu-west-1"
        K8S_DIR = "k8s"
        KUBECONFIG = "$WORKSPACE/kubeconfig"
    }

    stages {
        // Stage 1: Checkout Code
        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Amrfci11/library-repo.git',
                credentialsId: 'github-pip'
            }
        }

        // Stage 2: Build Docker Image
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        // Stage 3: Push to Docker Hub
        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-creds') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                    }
                }
            }
        }

        // Stage 4: Configure AWS/EKS Access
        stage('Configure AWS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        // Generate kubeconfig with 60s timeout
                        sh """
                            aws eks update-kubeconfig \
                              --name ${EKS_CLUSTER} \
                              --region ${AWS_REGION} \
                              --kubeconfig ${KUBECONFIG} \
                              --alias eks-${EKS_CLUSTER}
                            
                            export KUBECONFIG=${KUBECONFIG}
                            kubectl config set-cluster eks-${EKS_CLUSTER} \
                              --server="$(aws eks describe-cluster \
                                --name ${EKS_CLUSTER} \
                                --region ${AWS_REGION} \
                                --query 'cluster.endpoint' \
                                --output text)"
                            
                            kubectl config set-context --current \
                              --cluster=eks-${EKS_CLUSTER} \
                              --namespace=library-app
                        """
                    }
                }
            }
        }

        // Stage 5: Validate Cluster Access
        stage('Validate EKS Access') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        // Network pre-checks
                        sh """
                            echo "Testing EKS API connectivity..."
                            curl --connect-timeout 15 -v \
                              $(aws eks describe-cluster \
                                --name ${EKS_CLUSTER} \
                                --region ${AWS_REGION} \
                                --query 'cluster.endpoint' \
                                --output text)/api
                            
                            echo "Current kubectl context:"
                            kubectl config current-context
                            
                            echo "Cluster info:"
                            kubectl cluster-info
                        """
                    }
                }
            }
        }

        // Stage 6: Deploy to EKS
        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        // Apply manifests with retries
                        retry(3) {
                            timeout(time: 5, unit: 'MINUTES') {
                                sh """
                                    export KUBECONFIG=${KUBECONFIG}
                                    kubectl apply -f ${K8S_DIR}/namespace.yaml --validate=false
                                    kubectl apply -f ${K8S_DIR}/deployment.yaml
                                    kubectl apply -f ${K8S_DIR}/service.yaml
                                """
                            }
                        }

                        // Verify deployment
                        sh """
                            export KUBECONFIG=${KUBECONFIG}
                            kubectl rollout status deployment/library-app \
                              -n library-app \
                              --timeout=300s
                            
                            echo "Deployment successful! Service details:"
                            kubectl get svc -n library-app
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            // Clean up kubeconfig
            sh "rm -f ${KUBECONFIG} || true"
        }
        success {
            echo "Deployment to ${EKS_CLUSTER} completed successfully!"
        }
        failure {
            echo "Deployment failed. Check network connectivity and EKS cluster status."
            sh """
                echo "Debug info:"
                aws eks describe-cluster \
                  --name ${EKS_CLUSTER} \
                  --region ${AWS_REGION} \
                  --query 'cluster.{status:status,endpoint:endpoint}'
            """
        }
    }
}
