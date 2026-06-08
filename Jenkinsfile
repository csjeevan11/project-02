pipeline {
    agent slave-01

    parameters {
        string(name: 'APP_SERVER', defaultValue: '172.31.90.5', description: 'App Server IP')
        string(name: 'DOCKER_IMAGE', defaultValue: 'csjeevan/petclinic', description: 'Docker Image Name')
    }

    tools {
        maven 'Maven'
        jdk 'JDK17'
    }

    environment {
        DOCKER_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/csjeevan11/spring-petclinic.git'
            }
        }

        stage('Build Jar') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${params.DOCKER_IMAGE}:${DOCKER_TAG} .
                docker tag ${params.DOCKER_IMAGE}:${DOCKER_TAG} ${params.DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Login & Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push ${params.DOCKER_IMAGE}:${DOCKER_TAG}
                    docker push ${params.DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${params.APP_SERVER} '
                        docker pull ${params.DOCKER_IMAGE}:latest || true

                        docker stop petclinic || true
                        docker rm petclinic || true

                        docker run -d --name petclinic -p 8080:8080 \
			csjeevan/petclinic:${DOCKER_TAG}
                    '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Docker deployment successful"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
