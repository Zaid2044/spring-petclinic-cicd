pipeline {
    agent {
        label 'agent'
    }

    environment {
        IMAGE_NAME = 'Zaid2044/spring-petclinic'
        IMAGE_TAG  = 'latest'

        // Reduce Java memory usage on t3.micro
        MAVEN_OPTS = '-Xms128m -Xmx512m'
    }

    stages {

        stage('Checkout Source') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    sudo apt update

                    sudo apt install -y \
                        git \
                        docker.io \
                        openjdk-21-jdk \
                        unzip \
                        curl

                    sudo systemctl enable docker
                    sudo systemctl start docker

                    java -version
                    git --version
                    docker --version
                '''
            }
        }

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "===== Java ====="
                    java -version

                    echo "===== Maven ====="
                    ./mvnw -version

                    echo "===== Docker ====="
                    docker --version

                    echo "===== Git ====="
                    git --version

                    echo "===== Memory ====="
                    free -h
                '''
            }
        }

        stage('Build Application') {
            steps {
                echo "Building Application..."

                sh '''
                    ./mvnw clean package \
                        -DskipTests \
                        -Dcyclonedx.skip=true
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {

                withCredentials([
                    string(
                        credentialsId: 'sonarqube-token',
                        variable: 'SONAR_TOKEN'
                    )
                ]) {

                    withSonarQubeEnv('SonarQube') {

                        sh '''
                            ./mvnw sonar:sonar \
                                -DskipTests \
                                -Dcyclonedx.skip=true \
                                -Dsonar.projectKey=spring-petclinic \
                                -Dsonar.login=$SONAR_TOKEN
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

        stage('Build Docker Image') {
            steps {

                sh '''
                    docker build \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Docker Hub Login') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin
                    '''

                }
            }
        }

        stage('Push Docker Image') {

            steps {

                sh '''
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                '''

            }
        }
    }

    post {

        success {

            echo "===================================="
            echo "Pipeline completed successfully!"
            echo "Docker Image:"
            echo "${IMAGE_NAME}:${IMAGE_TAG}"
            echo "===================================="

        }

        failure {

            echo "===================================="
            echo "Pipeline Failed!"
            echo "===================================="

        }

        always {

            sh '''
                docker logout || true
            '''

        }
    }
}