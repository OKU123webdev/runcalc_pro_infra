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
                    PUBLIC_IP=$(aws ec2 describe-instances \
                      --filters "Name=tag:Name,Values=RunCalc_Prod" "Name=instance-state-name,Values=running" \
                      --query "Reservations[0].Instances[0].PublicIpAddress" \
                      --output text)

                    echo "Testing deployment on http://${PUBLIC_IP}"
                    curl --fail http://${PUBLIC_IP}
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