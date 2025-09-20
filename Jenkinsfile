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

// Jenkinsfile - production-ready (build -> push -> terraform)
// Assumptions:
// - Jenkins runs locally and has access to the host Docker daemon (docker socket mounted).
// - Jenkins has credentials created:
//    * GitHub PAT as "github-pat" (Username with password, username=GitHub-username, password=PAT)
//    * AWS keys as "aws-jenkins-creds" (Username with password, username=AWS_ACCESS_KEY_ID, password=AWS_SECRET_ACCESS_KEY)
// - Terraform and AWS CLI available in the Jenkins environment (see setup instructions below).
// - Terraform code under repo path: terraform/envs/<env>
// - ECR repo may be created in the pipeline if missing.

pipeline {
  agent any

  parameters {
    choice(name: 'ENV', choices: ['dev','test','staging','prod'], description: 'Target environment')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Image tag to push (defaults to BUILD_NUMBER if empty)')
  }

  environment {
    AWS_REGION = 'eu-west-2'              // change if you use another region
    ECR_REPO = 'senior-elearning-nginx'   // change if you use another repo name
    TF_DIR = "terraform/envs"             // root of terrraform env directories
  }

  stages {
    stage('Checkout pipeline repo') {
      steps {
        // this pipeline file is loaded from the job SCM; still keep a repo checkout for other resources
        echo "Using branch main"
        checkout([
          $class: 'GitSCM',
          branches: [[name: '*/main']],
          userRemoteConfigs: [[
            url: 'https://github.com/Geternn/senior-elearning-nginx.git',
            credentialsId: 'github-pat'
          ]]
        ])
      }
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
            # get account id
            AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --region ${AWS_REGION})
            ECR_URL=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

            # create repo if it does not exist
            if ! aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} >/dev/null 2>&1; then
              echo "ECR repo ${ECR_REPO} not found, creating..."
              aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION} >/dev/null
            else
              echo "ECR repo ${ECR_REPO} exists."
            fi

            # login to ECR
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
              # prepare terraform workspace
              terraform init -input=false
              terraform workspace list | grep -q "^${ENV}$" || terraform workspace new ${ENV} || terraform workspace select ${ENV}
              # run plan then apply
              terraform plan -input=false -var="image_tag=${IMAGE_TAG}" -var="aws_region=${AWS_REGION}"
              terraform apply -input=false -auto-approve -var="image_tag=${IMAGE_TAG}" -var="aws_region=${AWS_REGION}"
            '''
          }
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
}
