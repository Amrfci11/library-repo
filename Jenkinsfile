pipeline {
    agent any

    environment {
        // Configure your Docker Hub credentials (store them in Jenkins Credentials Manager)
        DOCKER_HUB_CREDS = credentials('docker-hub-creds') // Store these in Jenkins first!
        DOCKER_IMAGE = "amrsayed11/my-app" // Replace with your image name
        DOCKER_TAG = "latest"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/your-repo.git' // Update with your repo
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
    }
}
