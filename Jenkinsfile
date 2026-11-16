
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
        DOCKERHUB_REPO = 'ninande/my-web-app-new'

        ANSIBLE_HOST = '10.0.11.194'
        ANSIBLE_USER = 'ansadmin'
        ANSIBLE_INVENTORY = '/opt/ansible/my-web-app/inventory.ini'
        ANSIBLE_PLAYBOOK = '/opt/ansible/my-web-app/deploy.yml'

        COMPOSE_DIR = '/opt/my-web-app-new'
        NOTIFY_EMAIL = 'shreevdeshmukh@gmail.com'

        IMAGE_NAME = ''
        IMAGE_TAG = ''
    }

    stages {

        stage('1. Checkout') {
            steps {
                checkout scm

                sh '''
                    set -eu
                    git log -1 --oneline
                    test -f pom.xml
                    test -f Dockerfile
                '''
            }
        }

        stage('2. Build and Test') {
            steps {
                sh 'mvn -B clean verify'
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

        stage('3. Build Docker Image') {
            steps {
                script {
                    env.IMAGE_NAME = env.DOCKERHUB_REPO
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}"

                    echo "Building Docker Image: ${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    sh '''
                        set -eu

                        docker build \
                          -t "${IMAGE_NAME}:${IMAGE_TAG}" \
                          -t "${IMAGE_NAME}:latest" \
                          .
                    '''
                }
            }
        }

        stage('4. Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
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

        stage('5. Deploy with Docker Compose') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        cd "${COMPOSE_DIR}"

                        echo "$DH_TOKEN" | docker login \
                          --username "$DH_USER" \
                          --password-stdin

                        docker compose config -q
                        docker compose pull
                        docker compose up -d --remove-orphans
                        docker compose ps

                        docker logout || true
                    '''
                }
            }
        }

        stage('6. Deploy to Dev and Staging via Ansible') {
            steps {
                withEnv([
                    "IMAGE_NAME=${env.IMAGE_NAME}",
                    "IMAGE_TAG=${env.IMAGE_TAG}"
                ]) {
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

                            SSH_OPTS="-o BatchMode=yes -o IdentitiesOnly=yes -o StrictHostKeyChecking=yes -o ConnectTimeout=10"

                            ssh $SSH_OPTS \
                              -i "$ANSIBLE_KEY" \
                              -o UserKnownHostsFile="$KNOWN_HOSTS_FILE" \
                              "${ANSIBLE_USER}@${ANSIBLE_HOST}" \
                              'command -v ansible-playbook &&
                               test -r /opt/ansible/my-web-app/inventory.ini &&
                               test -r /opt/ansible/my-web-app/deploy.yml'

                            for ENVIRONMENT in dev staging; do

                                echo "Deploying to ${ENVIRONMENT}: ${IMAGE_NAME}:${IMAGE_TAG}"

                                ssh $SSH_OPTS \
                                  -i "$ANSIBLE_KEY" \
                                  -o UserKnownHostsFile="$KNOWN_HOSTS_FILE" \
                                  "${ANSIBLE_USER}@${ANSIBLE_HOST}" \
                                  "ansible-playbook \
                                   -i '${ANSIBLE_INVENTORY}' \
                                   '${ANSIBLE_PLAYBOOK}' \
                                   --limit '${ENVIRONMENT}' \
                                   -e 'image_name=${IMAGE_NAME}' \
                                   -e 'image_tag=${IMAGE_TAG}'"

                            done
                        '''
                    }
                }
            }
        }

        stage('7. Manual Approval - Production') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Approve production deployment of ${env.IMAGE_NAME}:${env.IMAGE_TAG}?",
                        ok: 'Approve Production',
                        submitter: 'admin'
                    )
                }
            }
        }

        stage('8. Deploy to Production via Ansible') {
            steps {
                withEnv([
                    "IMAGE_NAME=${env.IMAGE_NAME}",
                    "IMAGE_TAG=${env.IMAGE_TAG}"
                ]) {
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

                            SSH_OPTS="-o BatchMode=yes -o IdentitiesOnly=yes -o StrictHostKeyChecking=yes -o ConnectTimeout=10"

                            echo "Deploying ${IMAGE_NAME}:${IMAGE_TAG} to production"

                            ssh $SSH_OPTS \
                              -i "$ANSIBLE_KEY" \
                              -o UserKnownHostsFile="$KNOWN_HOSTS_FILE" \
                              "${ANSIBLE_USER}@${ANSIBLE_HOST}" \
                              'whoami &&
                               command -v ansible-playbook &&
                               test -r /opt/ansible/my-web-app/inventory.ini &&
                               test -r /opt/ansible/my-web-app/deploy.yml'

                            ssh $SSH_OPTS \
                              -i "$ANSIBLE_KEY" \
                              -o UserKnownHostsFile="$KNOWN_HOSTS_FILE" \
                              "${ANSIBLE_USER}@${ANSIBLE_HOST}" \
                              "ansible-playbook \
                               -i '${ANSIBLE_INVENTORY}' \
                               '${ANSIBLE_PLAYBOOK}' \
                               --limit prod \
                               -e 'image_name=${IMAGE_NAME}' \
                               -e 'image_tag=${IMAGE_TAG}'"

                            echo "Production deployment completed."
                        '''
                    }
                }
            }
        }
    }

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
                    body: "Job: ${env.JOB_NAME}\nBuild: #${env.BUILD_NUMBER}\nResult: ${currentBuild.currentResult}\nImage: ${env.IMAGE_NAME ?: 'Not built'}:${env.IMAGE_TAG ?: 'N/A'}\n\nConsole output:\n${env.BUILD_URL}console"
                )
            }
        }

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review the Jenkins console output.'
        }

        cleanup {
            sh 'docker image prune -f || true'
        }
    }
}
