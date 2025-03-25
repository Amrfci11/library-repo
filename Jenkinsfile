pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "amrsayed11/library-app"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        EKS_CLUSTER = "my-eks-cluster"
        AWS_REGION = "eu-west-1"
        K8S_DIR = "k8s"
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
                        kubectl config current-context
                    """
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
                    sh """
                        kubectl apply -f ${K8S_DIR}/namespace.yaml --validate=false
                        kubectl apply -f ${K8S_DIR}/deployment.yaml
                        kubectl apply -f ${K8S_DIR}/service.yaml
                        kubectl rollout status deployment/library-app -n library-app
                    """
                }
            }
        }
    }
}
