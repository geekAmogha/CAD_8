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
    stage('PrepareServer') {
      steps {
        sshagent (credentials: ['ec2_ssh_key']) {
          sh '''
          set -e
          echo "Preparing server directories and (optionally) Nginx root"
          ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'mkdir -p $RELEASE_DIR || true'
          # Attempt Nginx root update if Nginx is present and default site exists
          ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST 'if [ -d /etc/nginx ] && [ -f /etc/nginx/sites-enabled/default ]; then \
            if ! grep -q "/var/www/assignment8/current" /etc/nginx/sites-enabled/default; then \
              echo "Adjusting Nginx root to /var/www/assignment8/current"; \
              sudo sed -i "s#root .*#root /var/www/assignment8/current;#" /etc/nginx/sites-enabled/default || true; \
              sudo nginx -t && sudo systemctl reload nginx || true; \
            fi; \
          fi'
          '''
        }
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
        sh '''
        set -e
        echo "Running health check on $DEPLOY_HOST"
        STATUS=$(curl -o /tmp/page.html -s -w "%{http_code}" http://$DEPLOY_HOST/ || true)
        if ! grep -i "Cloud and DevOps Assignment 8" /tmp/page.html > /dev/null 2>&1; then
          echo "Health check content mismatch or page not served as expected."
          echo "Received HTTP status: $STATUS"
          echo "First 40 lines of page for diagnostics:";
          head -n 40 /tmp/page.html || true
          exit 1
        fi
        if [ "$STATUS" != "200" ]; then
          echo "Unexpected HTTP status: $STATUS";
          exit 1
        fi
        echo "Health check passed (status $STATUS, content OK)."
        '''
      }
    }
  }
  post {
    success { echo 'Deployment successful!' }
    failure { echo 'Deployment failed.' }
  }
}
