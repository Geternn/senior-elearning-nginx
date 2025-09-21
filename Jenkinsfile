pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Geternn/senior-elearning-nginx.git'
            }
        }

        stage('Verify Tools') {
            steps {
                sh 'aws --version'
                sh 'terraform -version'
            }
        }

        stage('Terraform Init') {
            steps {
                dir('infra') {
                    sh 'terraform init || true'
                }
            }
        }
    }
}
