pipeline {
    agent any
    parameters {
        string(name: 'image_tag', defaultValue: 'v1.0.5', description: 'Docker image tag to deploy')
    }
    stages {
        stage('Deploy via Ansible') {
            steps {
                sh """
                ansible-playbook -i inventory.ini playbook.yml --extra-vars "image_tag=${params.image_tag}"
                """
            }
        }
        stage('Verify Deployment') {
            steps {
                sh """
                curl -f http://devops-uws-b01813057.duckdns.org || echo "Deployment failed"
                """
            }
        }
    }
}