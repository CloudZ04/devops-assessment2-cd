pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0.5', description: 'Docker image tag to deploy')
    }

    environment {
        AWS_REGION = 'us-east-1'
        INSTANCE_TAG_NAME = 'Assignment2-CD-Prod'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/CloudZ04/devops-assessment2-cd.git'
            }
        }

        stage('Provision EC2') {
            steps {
                sh '''
                    ansible-playbook ansible/provision_staging_jenkins.yml
                '''
            }
        }

        stage('Resolve New Instance IP') {
            steps {
                script {
                    env.TARGET_IP = sh(
                        returnStdout: true,
                        script: '''
                            aws ec2 describe-instances \
                              --region "$AWS_REGION" \
                              --filters "Name=tag:Name,Values=$INSTANCE_TAG_NAME" "Name=instance-state-name,Values=running" \
                              --query "Reservations[].Instances[] | sort_by(@,&LaunchTime)[-1].PublicIpAddress" \
                              --output text
                        '''
                    ).trim()

                    if (!env.TARGET_IP || env.TARGET_IP == 'None' || env.TARGET_IP == 'null') {
                        error('Could not resolve target instance public IP')
                    }

                    echo "Deploy target IP: ${env.TARGET_IP}"
                }
            }
        }

        stage('Deploy Selected Tag') {
            steps {
                withCredentials([
                    string(credentialsId: 'ANSIBLE_VAULT_PASSWORD', variable: 'VAULT_PASS'),
                    sshUserPrivateKey(credentialsId: 'STAGING_SERVER_KEY', keyFileVariable: 'SSH_KEY')
                ]) {
                    sh '''
                        set -e
                        echo "$VAULT_PASS" > .vault_pass.txt

                        ansible-playbook ansible/playbook.yml \
                          -i "${TARGET_IP}," \
                          -u ubuntu \
                          --private-key "$SSH_KEY" \
                          --vault-password-file .vault_pass.txt \
                          --extra-vars "image_tag=${IMAGE_TAG}" \
                          --extra-vars "@ansible/vault.yml"

                        rm -f .vault_pass.txt
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    sleep 10
                    curl -f "http://${TARGET_IP}" >/dev/null
                '''
            }
        }
    }

    post {
        always {
            sh 'rm -f .vault_pass.txt || true'
        }
    }
}