pipeline {
    agent any

    tools {
        maven 'Maven'
        jdk 'JDK17'
    }

    environment {
        SONAR_HOST = "http://54.197.200.168:9000"
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

        stage('Build') {
            steps {
                sh '''
                mvn clean package -DskipTests -Dcheckstyle.skip=true
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
                    export PATH=$JAVA_HOME/bin:$PATH

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

        stage('Upload To Nexus') {
            steps {
                sh '''
                mvn deploy -DskipTests -Dcheckstyle.skip=true
                '''
            }
        }

    stage('Deploy To App Server') {
        steps {
            sshagent(['app-server-ssh']) {
                sh """
                ssh -o StrictHostKeyChecking=no ubuntu@${APP_SERVER} << EOF
                set -e

                cd /home/ubuntu
    
                echo "Stopping old application..."
                pkill -f spring-petclinic || true
    
                rm -f app.jar maven-metadata.xml
    
                NEXUS_URL=http://172.31.85.18:8081
    
                echo "Downloading metadata..."
                wget -q \$NEXUS_URL/repository/maven-snapshots/org/springframework/samples/spring-petclinic/4.0.0-SNAPSHOT/maven-metadata.xml
    
                echo "Checking metadata file..."
                cat maven-metadata.xml
    
                if [ ! -s maven-metadata.xml ]; then
                    echo "❌ metadata missing or empty"
                    exit 1
                fi

                echo "Extracting version safely..."
    
                VERSION=\$(grep '<value>' maven-metadata.xml | tail -1 | sed 's/.*<value>//;s/<\\/value>//')
    
                echo "Resolved VERSION: \$VERSION"
    
                if [ -z "\$VERSION" ]; then
                    echo "❌ VERSION extraction failed"
                    exit 1
                fi
    
                echo "Downloading JAR..."
                wget -O app.jar \
                \$NEXUS_URL/repository/maven-snapshots/org/springframework/samples/spring-petclinic/4.0.0-SNAPSHOT/spring-petclinic-\$VERSION.jar

                echo "Starting application..."
                nohup java -jar app.jar > app.log 2>&1 &
    
                sleep 10
    
                ps -ef | grep java | grep app.jar || true
    
                echo "DEPLOYMENT SUCCESS"
                EOF
                """
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
