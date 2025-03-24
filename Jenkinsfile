pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'mborham/jenkins-flask:latest' 
        DOCKER_HUB_CREDENTIALS = 'docker-hub-credentials' 
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/MuhammadBorham/newfalsk.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${env.DOCKER_IMAGE}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', "${env.DOCKER_HUB_CREDENTIALS}") {
                        docker.image("${env.DOCKER_IMAGE}").push()
                    }
                }
            }
        }
    }
}
