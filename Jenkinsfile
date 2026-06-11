pipeline {
    agent any

    parameters {
        string(name: 'APP_SERVER_IP', defaultValue: '172.31.90.5', description: 'App server IP')
        string(name: 'APP_PORT', defaultValue: '8080', description: 'App port')
        string(name: 'NEXUS_IP', defaultValue: '172.31.85.18', description: 'Nexus server IP')
        string(name: 'SONAR_HOST', defaultValue: 'http://18.205.159.66:9000', description: 'SonarQube URL')
    }

    tools {
        maven 'Maven'
        jdk 'JDK17'
    }

    environment {
        IMAGE_NAME = "jeevan11cs/spring-petclinic"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        NEXUS_URL  = "http://${params.NEXUS_IP}:8081"
        SONAR_HOST = "${params.SONAR_HOST}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/csjeevan11/spring-petclinic.git'
            }
        }

        stage('Build Maven') {
            steps {
                sh '''
                    export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
                    export PATH=$JAVA_HOME/bin:$PATH

                    java -version
                    mvn -version

                    mvn clean package -DskipTests -Dcheckstyle.skip=true
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn sonar:sonar \
                            -Dsonar.projectKey=petclinic \
                            -Dsonar.projectName=petclinic
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
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
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy to App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${params.APP_SERVER_IP} '
                            set -e

                            IMAGE_NAME=${IMAGE_NAME}
                            IMAGE_TAG=${IMAGE_TAG}

                            echo "Pulling image..."
                            docker pull \$IMAGE_NAME:\$IMAGE_TAG

                            echo "Stopping old container..."
                            docker stop petclinic || true
                            docker rm petclinic || true

                            echo "Starting new container..."
                            docker run -d \
                              --name petclinic \
                              -p ${params.APP_PORT}:8080 \
                              --restart unless-stopped \
                              \$IMAGE_NAME:\$IMAGE_TAG

                            echo "Deployment completed successfully"
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
