
pipeline {
    agent {
        label 'dockerhost'
    }

    triggers {
        githubPush()
    }

    options {
        timestamps()
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        DOCKERHUB_CREDENTIALS = 'dockerhub-creds'
        DOCKERHUB_REPO = 'ninande/my-web-app-new-mon'

        ANSIBLE_HOST = '10.0.11.194'
        ANSIBLE_USER = 'ansadmin'
        ANSIBLE_DIR = '/opt/monitoring/my-web-app'
        ANSIBLE_INVENTORY = '/opt/monitoring/my-web-app/inventory.ini'
        ANSIBLE_PLAYBOOK = '/opt/monitoring/my-web-app/deploy.yml'

        NOTIFY_EMAIL = 'shreevdeshmukh@gmail.com'

        IMAGE_NAME = 'ninande/my-web-app-new-mon'
        IMAGE_TAG = ''
    }

    stages {

        // STAGE 1: CHECKOUT SOURCE CODE

        stage('1. Checkout') {
            steps {
                checkout scm

                sh '''
                    set -eu

                    echo "Checking source code"

                    git log -1 --oneline

                    test -f pom.xml
                    test -f Dockerfile
                '''
            }
        }

        // STAGE 2: MAVEN BUILD AND TEST

        stage('2. Build and Test') {
            steps {
                sh '''
                    set -eu
                    mvn -B clean verify
                '''
            }

            post {
                always {
                    junit(
                        testResults: 'target/surefire-reports/TEST-*.xml',
                        allowEmptyResults: true
                    )
                }
            }
        }

        // STAGE 3: BUILD DOCKER IMAGE

      stage('3. Build Docker Image') {
    steps {
        script {
            env.IMAGE_NAME = 'ninande/my-web-app-new-mon'
            env.IMAGE_TAG = "${env.BUILD_NUMBER}"

            echo "Building Docker Image: ${env.IMAGE_NAME}:${env.IMAGE_TAG}"

            sh """
                set -eu

                docker build \
                  -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} \
                  -t ${env.IMAGE_NAME}:latest \
                  .
            """
        }
    }
}

        // STAGE 4: PUSH IMAGE TO DOCKER HUB

        stage('4. Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKERHUB_CREDENTIALS}",
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        echo "$DH_TOKEN" | docker login \
                            --username "$DH_USER" \
                            --password-stdin

                        docker push "${IMAGE_NAME}:${IMAGE_TAG}"
                        docker push "${IMAGE_NAME}:latest"

                        docker logout || true
                    '''
                }
            }
        }

        // STAGE 5: VERIFY ANSIBLE CONNECTIVITY

        stage('5. Ansible Connectivity') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-control-ssh-key',
                        keyFileVariable: 'ANSIBLE_KEY'
                    ),
                    file(
                        credentialsId: 'ansible-known-hosts',
                        variable: 'KNOWN_HOSTS_FILE'
                    )
                ]) {
                    sh '''
                        set -eu

                        SSH_OPTS="-o BatchMode=yes \
                        -o IdentitiesOnly=yes \
                        -o StrictHostKeyChecking=yes \
                        -o ConnectTimeout=10"

                        echo "Connecting to Ansible Control Server"

                        ssh $SSH_OPTS \
                            -i "$ANSIBLE_KEY" \
                            -o UserKnownHostsFile="$KNOWN_HOSTS_FILE" \
                            "${ANSIBLE_USER}@${ANSIBLE_HOST}" \
                            "whoami &&
                             command -v ansible-playbook &&
                             test -r '${ANSIBLE_INVENTORY}' &&
                             test -r '${ANSIBLE_PLAYBOOK}'"

                        echo "Testing all three target servers"

                        ssh $SSH_OPTS \
                            -i "$ANSIBLE_KEY" \
                            -o UserKnownHostsFile="$KNOWN_HOSTS_FILE" \
                            "${ANSIBLE_USER}@${ANSIBLE_HOST}" \
                            "cd '${ANSIBLE_DIR}' &&
                             ansible all -i inventory.ini -m ping"
                    '''
                }
            }
        }

        // STAGE 6: DEPLOY TO TWO APP SERVERS AND NGINX LB

        stage('6. Deploy using Ansible') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-control-ssh-key',
                        keyFileVariable: 'ANSIBLE_KEY'
                    ),
                    file(
                        credentialsId: 'ansible-known-hosts',
                        variable: 'KNOWN_HOSTS_FILE'
                    )
                ]) {
                    sh '''
                        set -eu

                        SSH_OPTS="-o BatchMode=yes \
                        -o IdentitiesOnly=yes \
                        -o StrictHostKeyChecking=yes \
                        -o ConnectTimeout=10"

                        echo "Starting Ansible deployment"

                        echo "Docker Image: ${IMAGE_NAME}:${IMAGE_TAG}"

                        ssh $SSH_OPTS \
                            -i "$ANSIBLE_KEY" \
                            -o UserKnownHostsFile="$KNOWN_HOSTS_FILE" \
                            "${ANSIBLE_USER}@${ANSIBLE_HOST}" \
                            "cd '${ANSIBLE_DIR}' &&
                             ansible-playbook \
                             -i inventory.ini \
                             deploy.yml \
                             -e 'image_name=${IMAGE_NAME}' \
                             -e 'image_tag=${IMAGE_TAG}'"

                        echo "Deployment completed"
                    '''
                }
            }
        }

        // STAGE 7: VERIFY DEPLOYMENT

        stage('7. Verify Deployment') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-control-ssh-key',
                        keyFileVariable: 'ANSIBLE_KEY'
                    ),
                    file(
                        credentialsId: 'ansible-known-hosts',
                        variable: 'KNOWN_HOSTS_FILE'
                    )
                ]) {
                    sh '''
                        set -eu

                        SSH_OPTS="-o BatchMode=yes \
                        -o IdentitiesOnly=yes \
                        -o StrictHostKeyChecking=yes"

                        echo "Verifying application deployment"

                        ssh $SSH_OPTS \
                            -i "$ANSIBLE_KEY" \
                            -o UserKnownHostsFile="$KNOWN_HOSTS_FILE" \
                            "${ANSIBLE_USER}@${ANSIBLE_HOST}" \
                            "cd '${ANSIBLE_DIR}' &&
                             ansible load_balancer -i inventory.ini \
                             -m shell -a 'systemctl is-active nginx' &&
                             ansible app_servers -i inventory.ini \
                             -m shell -a 'docker ps'"

                        echo "Deployment verification completed"
                    '''
                }
            }
        }
    }

    // POST BUILD ACTIONS

    post {

        always {
            archiveArtifacts(
                artifacts: 'reports/**',
                allowEmptyArchive: true,
                fingerprint: true
            )

            script {
                echo "Pipeline result: ${currentBuild.currentResult}"

                emailext(
                    to: "${NOTIFY_EMAIL}",
                    subject: "${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
Job: ${env.JOB_NAME}

Build Number: ${env.BUILD_NUMBER}

Result: ${currentBuild.currentResult}

Docker Image: ${env.IMAGE_NAME ?: 'Not built'}:${env.IMAGE_TAG ?: 'N/A'}

Build URL:
${env.BUILD_URL}

Console Output:
${env.BUILD_URL}console
"""
                )
            }
        }

        success {
            echo 'Deployment completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins console output.'
        }

        cleanup {
            sh 'docker image prune -f || true'
        }
    }
}
