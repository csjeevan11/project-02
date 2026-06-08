pipeline {
    agent { label 'web-builder' }

    parameters {

        string(name: 'APP_SERVER', defaultValue: '172.31.90.5', description: 'Target App Server IP')

        string(name: 'APP_PORT', defaultValue: '8080', description: 'Application Port')

        string(name: 'DOCKER_IMAGE', defaultValue: 'csjeevan/petclinic', description: 'Docker Image Name')

        string(name: 'DOCKER_TAG', defaultValue: "${BUILD_NUMBER}", description: 'Image Tag')

    }

    tools {
        maven 'Maven'
        jdk 'JDK17'
    }

    environment {
        IMAGE = "${params.DOCKER_IMAGE}"
        TAG = "${params.DOCKER_TAG}"
        FULL_IMAGE = "${params.DOCKER_IMAGE}:${params.DOCKER_TAG}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/csjeevan11/spring-petclinic.git'
            }
        }

        stage('Build Application') {
            steps {
                sh """
                    mvn clean package -DskipTests
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${FULL_IMAGE} .
                    docker tag ${FULL_IMAGE} ${IMAGE}:latest
                """
            }
        }

        stage('Docker Login & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${FULL_IMAGE}
                        docker push ${IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${params.APP_SERVER} '
                            set -e

                            echo "Pulling latest image..."
                            docker pull ${IMAGE}:latest

                            echo "Stopping old container..."
                            docker stop petclinic || true
                            docker rm petclinic || true

                            echo "Starting new container..."
                            docker run -d \
                              --name petclinic \
                              -p ${params.APP_PORT}:8080 \
                              ${IMAGE}:latest

                            echo "Deployment completed"
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS: Application deployed successfully"
        }

        failure {
            echo "Pipeline FAILED: Check logs"
        }

        always {
            cleanWs()
        }
    }
}
