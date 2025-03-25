pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "amrsayed11/library-app"  // Change to your desired image name
        DOCKER_TAG = "latest"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/Amrfci11/library-repo.git',
                credentialsId: 'github-pip'  // Your existing GitHub credentials
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
                    docker.withRegistry('https://registry.hub.docker.com', 'dckr_pat_maUo-N1o5befj3AGsV-Uy5PrkaA') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                    }
                }
            }
        }
    }
}
