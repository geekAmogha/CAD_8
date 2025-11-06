pipeline {
  agent any
  environment {
    DEPLOY_HOST = 'ec2-43-204-108-165.ap-south-1.compute.amazonaws.com'        // Replace with actual EC2 public DNS or IP
    DEPLOY_USER = 'ubuntu'                   // Or 'ubuntu' depending on AMI
    RELEASE_DIR = '/var/www/assignment8/releases'
    CURRENT_LINK = '/var/www/assignment8/current'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '15'))
    timestamps()
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Validate') {
      steps {
        echo 'Add HTML/CSS linting tools here if desired.'
      }
    }
    stage('Archive') {
      steps {
        archiveArtifacts artifacts: 'index.html,style.css', fingerprint: true
      }
    }
    stage('Deploy') {
      steps {
        sshagent (credentials: ['ec2_ssh_key']) {
          sh '''
          set -e
          TS=$(date +%s)
          TARGET="$RELEASE_DIR/$TS"
          echo "Deploying to $TARGET on $DEPLOY_HOST"
          ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST "mkdir -p $RELEASE_DIR && mkdir -p $TARGET"
          scp -o StrictHostKeyChecking=no index.html style.css $DEPLOY_USER@$DEPLOY_HOST:$TARGET/
          ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST "ln -sfn $TARGET $CURRENT_LINK"
          '''
        }
      }
    }
    stage('Health Check') {
      steps {
        sh 'curl -fsS http://$DEPLOY_HOST/ | grep -i "Cloud and DevOps Assignment 8"'
      }
    }
  }
  post {
    success { echo 'Deployment successful!' }
    failure { echo 'Deployment failed.' }
  }
}
