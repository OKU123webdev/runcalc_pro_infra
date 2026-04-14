pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0.1', description: 'Docker image tag to deploy')
    }

    stages {
        stage('Get Code') {
            steps {
                checkout scm
            }
        }

        stage('Provision EC2 and Configure Server') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'PROD_SERVER_KEY', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER'),
                    string(credentialsId: 'DUCKDNS_TOKEN', variable: 'DUCKDNS_TOKEN'),
                    string(credentialsId: 'ANSIBLE_VAULT_PASSWORD', variable: 'VAULT_PASSWORD')
                ]) {
                    sh '''
                        printf "%s" "$VAULT_PASSWORD" > .vault_pass
                        chmod 600 .vault_pass

                        ansible-playbook provision_production.yml \
                          --private-key "$SSH_KEY_FILE" \
                          --extra-vars "duckdns_api_token=$DUCKDNS_TOKEN" \
                          --vault-password-file .vault_pass
                    '''
                }
            }
        }

        stage('Deploy Selected Image') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'PROD_SERVER_KEY', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER'),
                    string(credentialsId: 'ANSIBLE_VAULT_PASSWORD', variable: 'VAULT_PASSWORD')
                ]) {
                    sh '''
                        printf "%s" "$VAULT_PASSWORD" > .vault_pass
                        chmod 600 .vault_pass

                        ansible-playbook deploy_application_to_production.yml \
                          --private-key "$SSH_KEY_FILE" \
                          --extra-vars "image_tag=${IMAGE_TAG}" \
                          --vault-password-file .vault_pass
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Testing deployment via DuckDNS"
                    for i in {1..12}; do
                        if curl --fail --silent http://b01663625-staging.duckdns.org > /dev/null; then
                            echo "Deployment verified successfully."
                            exit 0
                        fi
                        echo "Attempt $i failed. Waiting 10 seconds..."
                        sleep 10
                    done
                    echo "Deployment verification failed after multiple attempts."
                    exit 1
                '''
            }
        }
    }

    post {
        always {
            sh 'rm -f .vault_pass || true'
        }
    }
}