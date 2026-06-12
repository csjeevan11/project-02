pipeline {
    agent { label 'node-01' }

    parameters {
        string(name: 'SONAR_HOST', defaultValue: 'http://18.205.106.241:9000', description: 'SonarQube URL')
    }

    tools {
        maven 'Maven'
        jdk 'JDK17'
    }

    environment {
        IMAGE_NAME = "jeevan11cs/spring-petclinic"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        SONAR_HOST = "${params.SONAR_HOST}"
        K8S_NAMESPACE = "petclinic"
        HELM_RELEASE  = "petclinic"
        DOCKER_BUILDKIT = "1"
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${PATH}"
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
        
        stage('Docker Cleanup') {
            steps {
                sh '''
                    docker image prune -f || true
                '''
            }
        }

        stage('Helm Deploy To Kubernetes') {
            steps {
                sh '''
                    kubectl config current-context
                    
                    helm upgrade --install ${HELM_RELEASE} \
                    ./helm-charts/spring-petclinic \
                    -n ${K8S_NAMESPACE} \
                    --set image.repository=${IMAGE_NAME} \
                    --set image.tag=${IMAGE_TAG}

                    kubectl rollout status deployment/petclinic \
                    -n ${K8S_NAMESPACE} \
                    --timeout=300s

                    kubectl get deployment petclinic \
                    -n ${K8S_NAMESPACE} \
                    -o=jsonpath='{.spec.template.spec.containers[0].image}'
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "===== PODS ====="
                    kubectl get pods -n ${K8S_NAMESPACE}

                    echo "===== SERVICE ====="
                    kubectl get svc -n ${K8S_NAMESPACE}

                    echo "===== INGRESS ====="
                    kubectl get ingress -n ${K8S_NAMESPACE}

                    echo "===== HELM RELEASE ====="
                    helm list -n ${K8S_NAMESPACE}
                '''   
            }
        }
    }

    post {
        success {
            echo "SUCCESS: Build ${BUILD_NUMBER} deployed to Kubernetes"
            sh '''
                helm history petclinic -n petclinic
                helm status petclinic -n petclinic
            '''
        }
        failure {
            echo "Pipeline FAILED: Check logs"
        }
        always {
            cleanWs()
        }
    }
}
