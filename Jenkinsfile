pipeline {
    agent { label 'node-01' }

    parameters {
        string(name: 'APP_SERVER_IP', defaultValue: '172.31.90.5', description: 'App server IP')
        string(name: 'NEXUS_IP', defaultValue: '172.31.85.18', description: 'Nexus server IP')
        string(name: 'SONAR_HOST', defaultValue: 'http://18.205.159.66:9000')
    }

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
                    export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
                    export PATH=$JAVA_HOME/bin:$PATH
                    java -version
                    mvn -version
                    mvn clean package -DskipTests -Dcheckstyle.skip=true
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
