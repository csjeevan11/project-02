pipeline {
    agent any

    tools {
        maven 'Maven'
        jdk 'JDK17'
    }

    environment {
        SONAR_HOST = "172.31.88.60:9000"
        NEXUS_URL  = "http://172.31.85.18:8081"
        APP_SERVER = "172.31.90.5"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/csjeevan11/spring-petclinic.git'
            }
        }
        stage('Debug Environment') {
            steps {
                sh '''
                whoami
                java -version
                javac -version
                mvn -version
                echo JAVA_HOME=$JAVA_HOME
                which java
                which javac
                ls -l $(which java)
                ls -l $(which javac)
                '''
            }
        }
        stage('Build') {
            steps {
                sh '''
                export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
                export PATH=$JAVA_HOME/bin:$PATH
                java -version
                javac -version
                mvn clean package \
                -DskipTests \
                -Dcheckstyle.skip=true
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('SonarQube') {

                    withCredentials([
                        string(
                            credentialsId: 'sonar-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {

                        sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=petclinic \
                        -Dsonar.host.url=http://$SONAR_HOST \
                        -Dsonar.token=$SONAR_TOKEN \
                        -Dcheckstyle.skip=true
                        '''
                    }
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

        stage('Upload To Nexus') {
            steps {

                sh '''
                mvn deploy \
                -DskipTests \
                -Dcheckstyle.skip=true
                '''
            }
        }

        stage('Deploy To App Server') {

            steps {

                sshagent(['app-server-ssh']) {

                    sh '''
                    ssh -o StrictHostKeyChecking=no ubuntu@$APP_SERVER "

                    pkill -f spring-petclinic || true

                    rm -f app.jar

                    wget -O app.jar \
                    $NEXUS_URL/repository/maven-releases/org/springframework/samples/spring-petclinic/4.0.0-SNAPSHOT/spring-petclinic-4.0.0-SNAPSHOT.jar

                    nohup java -jar app.jar \
                    > app.log 2>&1 &

                    "
                    '''
                }
            }
        }
    }

    post {

        success {
            echo 'Application deployed successfully'
        }

        failure {
            echo 'Pipeline failed'
        }
    }
}
