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
                    env.TARGET_PUBLIC_IP = sh(
                        returnStdout: true,
                        script: '''
                            aws ec2 describe-instances \
                            --region "$AWS_REGION" \
                            --filters "Name=tag:Name,Values=$INSTANCE_TAG_NAME" "Name=instance-state-name,Values=running" \
                            --query "Reservations[].Instances[] | sort_by(@,&LaunchTime)[-1].PublicIpAddress" \
                            --output text
                        '''
                    ).trim()

                    env.TARGET_PRIVATE_IP = sh(
                        returnStdout: true,
                        script: '''
                            aws ec2 describe-instances \
                            --region "$AWS_REGION" \
                            --filters "Name=tag:Name,Values=$INSTANCE_TAG_NAME" "Name=instance-state-name,Values=running" \
                            --query "Reservations[].Instances[] | sort_by(@,&LaunchTime)[-1].PrivateIpAddress" \
                            --output text
                        '''
                    ).trim()

                    if (!env.TARGET_PUBLIC_IP || env.TARGET_PUBLIC_IP == 'None' || env.TARGET_PUBLIC_IP == 'null') {
                        error('Could not resolve target instance public IP')
                    }

                    if (!env.TARGET_PRIVATE_IP || env.TARGET_PRIVATE_IP == 'None' || env.TARGET_PRIVATE_IP == 'null') {
                        error('Could not resolve target instance private IP')
                    }

                    echo "Deploy target public IP: ${env.TARGET_PUBLIC_IP}"
                    echo "Deploy target private IP: ${env.TARGET_PRIVATE_IP}"
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
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ls -l "$SSH_KEY"
                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=10 ubuntu@"${TARGET_PRIVATE_IP}" "echo ok-from-jenkins-cred"
                        ansible-playbook ansible/playbook.yml \
                          -i "${TARGET_PRIVATE_IP}," \
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
                    curl -f "http://${TARGET_PUBLIC_IP}" >/dev/null
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