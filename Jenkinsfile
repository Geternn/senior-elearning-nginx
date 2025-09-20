pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Geternn/senior-elearning-nginx.git'
            }
        }

        stage('Test') {
            steps {
                echo 'Repo checkout successful!'
                sh 'ls -la'
            }
        }
    }
}
