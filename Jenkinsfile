pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_HOST   = '10.0.1.5'
        DEPLOY_DIR = '/opt/healthcare'
        IMAGE_NAME = 'healthcare-frontend'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Build Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        IMAGE="$DOCKERHUB_USER/$IMAGE_NAME"

                        docker build \
                          -t "$IMAGE:$BUILD_NUMBER" \
                          -t "$IMAGE:latest" .

                        echo "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USER" \
                          --password-stdin

                        docker push "$IMAGE:$BUILD_NUMBER"
                        docker push "$IMAGE:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Application Server') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sshagent(credentials: ['app-server-ssh']) {
                        sh '''
                            set -eu

                            scp -o StrictHostKeyChecking=accept-new \
                              docker-compose.yml \
                              ec2-user@$APP_HOST:$DEPLOY_DIR/docker-compose.yml

                            ssh -o StrictHostKeyChecking=accept-new \
                              ec2-user@$APP_HOST \
                              "DOCKERHUB_USER='$DOCKERHUB_USER' \
                               IMAGE_TAG='$BUILD_NUMBER' \
                               docker compose -f '$DEPLOY_DIR/docker-compose.yml' \
                               pull frontend"

                            ssh -o StrictHostKeyChecking=accept-new \
                              ec2-user@$APP_HOST \
                              "DOCKERHUB_USER='$DOCKERHUB_USER' \
                               IMAGE_TAG='$BUILD_NUMBER' \
                               docker compose -f '$DEPLOY_DIR/docker-compose.yml' \
                               up -d --remove-orphans"

                            ssh -o StrictHostKeyChecking=accept-new \
                              ec2-user@$APP_HOST \
                              "DOCKERHUB_USER='$DOCKERHUB_USER' \
                               IMAGE_TAG='$BUILD_NUMBER' \
                               docker compose -f '$DEPLOY_DIR/docker-compose.yml' \
                               ps"
                        '''
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sshagent(credentials: ['app-server-ssh']) {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKERHUB_USER',
                            passwordVariable: 'DOCKERHUB_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -eu

                            echo "Checking application containers..."

                            ssh -o StrictHostKeyChecking=accept-new \
                              ec2-user@$APP_HOST \
                              "DOCKERHUB_USER='$DOCKERHUB_USER' \
                               IMAGE_TAG='$BUILD_NUMBER' \
                               docker compose -f '$DEPLOY_DIR/docker-compose.yml' \
                               ps"

                            echo "Checking HTTP response..."

                            curl --fail --retry 5 --retry-delay 3 \
                              "http://$APP_HOST/"

                            echo "Deployment verification completed."
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Healthcare deployment succeeded: build ${BUILD_NUMBER}"
        }

        failure {
            echo 'Deployment failed. Review the Jenkins console output.'
        }
    }
}
