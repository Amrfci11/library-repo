pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "amrsayed11/library-app"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        EKS_CLUSTER = "my-eks-cluster"  // Update with your cluster name
        AWS_REGION = "us-east-1"        // Update your region
        K8S_DIR = "k8s"                 // Path to your manifests
    }

    stages {
        // Stage 1: Checkout Code with K8s manifests
        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Amrfci11/library-repo.git',
                credentialsId: 'github-cred'
            }
        }

        // Stage 2: Build & Push Docker Image (optional)
        stage('Build Docker Image') {
            when {
                expression { return fileExists('Dockerfile') }
            }
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-creds') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                    }
                }
            }
        }

        // Stage 3: Configure AWS/K8s Access
        stage('Configure AWS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh """
                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER}
                    """
                }
            }
        }

        // Stage 4: Deploy to EKS
        stage('Deploy K8s Manifests') {
            steps {
                script {
                    // Apply all manifests in k8s directory
                    sh """
                        kubectl apply -f ${K8S_DIR}/namespace.yaml
                        kubectl apply -f ${K8S_DIR}/deployment.yaml
                        kubectl apply -f ${K8S_DIR}/service.yaml
                    """

                    // Update image if Docker stage was executed
                    if (fileExists('Dockerfile')) {
                        sh """
                            kubectl set image deployment/library-app \
                              library-app=${DOCKER_IMAGE}:${DOCKER_TAG} \
                              -n library-app
                        """
                    }

                    // Verify deployment
                    sh """
                        kubectl rollout status deployment/library-app -n library-app
                        kubectl get svc -n library-app
                    """
                }
            }
        }
    }

    post {
        success {
            slackSend channel: '#deployments',
                     message: "SUCCESS: ${DOCKER_IMAGE}:${DOCKER_TAG} deployed to ${EKS_CLUSTER}"
        }
        failure {
            slackSend channel: '#deployments',
                     message: "FAILED: Deployment to ${EKS_CLUSTER} (Build ${env.BUILD_NUMBER})"
        }
    }
}
