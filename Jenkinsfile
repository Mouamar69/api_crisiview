pipeline {
    agent { label 'agent_node' }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Tests') {
            steps {
                sh '''
                    docker run -d --name mysql-test \
                      --network devops \
                      -e MYSQL_ROOT_PASSWORD=root \
                      -e MYSQL_DATABASE=incident_db \
                      mysql:8
                    until docker exec mysql-test mysql -u root -proot -e "SELECT 1" 2>/dev/null; do sleep 3; done
                    DB_HOST=mysql-test DB_PORT=3306 npm test -- --coverage --forceExit
                '''
            }
            post {
                always {
                    sh 'docker stop mysql-test || true'
                    sh 'docker rm mysql-test || true'
                    archiveArtifacts artifacts: 'coverage/**', allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.host.url=http://sonarqube_container:9000 \
                              -Dsonar.token=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Package') {
            steps {
                sh 'docker build -t crisisview-api:$BUILD_NUMBER -t crisisview-api:latest .'
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker tag crisisview-api:latest $DOCKER_USER/crisisview-api:latest
                        docker tag crisisview-api:$BUILD_NUMBER $DOCKER_USER/crisisview-api:$BUILD_NUMBER
                        docker push $DOCKER_USER/crisisview-api:latest
                        docker push $DOCKER_USER/crisisview-api:$BUILD_NUMBER
                    '''
                }
            }
        }

        stage('Deploy Staging') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'DOCKER_USER=$DOCKER_USER docker compose up -d --pull always api db'
                }
            }
        }
    }

    post {
        failure {
            echo 'Pipeline en echec. Rollback : docker compose up -d --no-build api'
        }
        always {
            archiveArtifacts artifacts: 'coverage/**', allowEmptyArchive: true
        }
    }
}
