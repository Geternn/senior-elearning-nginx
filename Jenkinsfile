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

parameters {
    choice(name: 'ENV', choices: ['dev','test','staging','prod'], description: 'Target environment')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Image tag to push (defaults to BUILD_NUMBER if empty)')
}

environment {
    AWS_REGION = 'eu-west-2'             // change to your region
    ECR_REPO = 'senior-elearning-nginx'  // change if different
    TF_DIR = "terraform/envs"            // terraform environment folders
}

stage('Set image tag') {
    steps {
        script {
            if (!params.IMAGE_TAG?.trim()) {
                env.IMAGE_TAG = "${env.BUILD_NUMBER}"
            } else {
                env.IMAGE_TAG = params.IMAGE_TAG
            }
            echo "IMAGE_TAG = ${env.IMAGE_TAG}"
        }
    }
}

stage('Build Docker image') {
    steps {
        sh '''
            echo "Building docker image..."
            docker build --pull -t ${ECR_REPO}:${IMAGE_TAG} ./docker
        '''
    }
}

stage('Prepare ECR and Login') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'aws-jenkins-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
            sh '''
                set -e
                AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --region ${AWS_REGION})
                ECR_URL=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                if ! aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} >/dev/null 2>&1; then
                  echo "ECR repo ${ECR_REPO} not found, creating..."
                  aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION} >/dev/null
                else
                  echo "ECR repo ${ECR_REPO} exists."
                fi

                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URL}
            '''
        }
    }
}

stage('Tag & Push') {
    steps {
        sh '''
            AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --region ${AWS_REGION})
            ECR_URL=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
            docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URL}/${ECR_REPO}:${IMAGE_TAG}
            echo "Pushing image ${ECR_URL}/${ECR_REPO}:${IMAGE_TAG} ..."
            docker push ${ECR_URL}/${ECR_REPO}:${IMAGE_TAG}
        '''
    }
}

stage('Terraform init & apply') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'aws-jenkins-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
            dir("${TF_DIR}/${params.ENV}") {
                sh '''
                    set -e
                    terraform init -input=false
                    terraform workspace list | grep -q "^${ENV}$" || terraform workspace new ${ENV} || terraform workspace select ${ENV}
                    terraform plan -input=false -var="image_tag=${IMAGE_TAG}" -var="aws_region=${AWS_REGION}"
                    terraform apply -input=false -auto-approve -var="image_tag=${IMAGE_TAG}" -var="aws_region=${AWS_REGION}"
                '''
            }
        }
    }
}

post {
    success {
        echo "Pipeline succeeded. Image tag: ${IMAGE_TAG}"
    }
    failure {
        echo "Pipeline failed. Check console output for errors."
    }
}


