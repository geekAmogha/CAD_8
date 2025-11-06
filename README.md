# Cloud & DevOps Assignment 8

Modern static HTML/CSS page demonstrating a CI/CD pipeline using **Jenkins** deploying to an **AWS EC2** instance.

## Project Contents

- `index.html` – Main landing page (accessible, responsive, semantic structure).
- `style.css` – Visual design: gradient background, glass effect panels, responsive typography.
- `Jenkinsfile` – Declarative pipeline (to be added) for automated validation and deployment.

## CI/CD Pipeline Overview

Triggered on pushes to `main` (or PRs):

1. Checkout source code.
2. Validation stage: basic HTML/CSS lint (e.g., `htmlhint`, `stylelint`) – optional.
3. Archive artifact (HTML/CSS) for traceability.
4. Deploy to EC2:
	 - Use SSH credentials stored in Jenkins (`ec2_ssh_key`).
	 - Secure copy updated files to `/var/www/assignment8/releases/<build-number>`.
	 - Update a symlink `/var/www/assignment8/current` -> new release.
5. Health check: curl public URL and fail if not HTTP 200.

## AWS EC2 Setup (Prerequisites)

1. Launch an EC2 instance (Amazon Linux 2023 or Ubuntu LTS) with security group allowing HTTP (80) + SSH (22).
2. Install a web server (e.g., Nginx):
	 ```bash
	 sudo apt update && sudo apt install -y nginx
	 # or Amazon Linux: sudo dnf install -y nginx
	 sudo systemctl enable --now nginx
	 ```
3. Create directory structure:
	 ```bash
	 sudo mkdir -p /var/www/assignment8/releases
	 sudo chown -R ec2-user:ec2-user /var/www/assignment8
	 ```
4. Point Nginx root to the symlink (example server block):
	 ```nginx
	 server {
		 listen 80;
		 server_name _;
		 root /var/www/assignment8/current;
		 index index.html;
		 location / { try_files $uri $uri/ =404; }
	 }
	 ```
5. Reload Nginx after config change: `sudo nginx -t && sudo systemctl reload nginx`.

## Jenkins Configuration

Install plugins (optional but helpful):
- Pipeline
- SSH Agent
- HTML Publisher (if you want to publish the page as a build artifact preview)

Add Credentials:
- SSH private key: ID `ec2_ssh_key` (type: SSH Username with private key).

Global Tools (optional):
- Node.js (if you decide to add linters via npm)

## Example Jenkinsfile (Pipeline)

Will be added as `Jenkinsfile` in the repo:
```groovy
pipeline {
	agent any
	environment {
		DEPLOY_HOST = 'YOUR.EC2.PUBLIC.DNS'
		DEPLOY_USER = 'ec2-user' // or ubuntu
		RELEASE_DIR = '/var/www/assignment8/releases'
		CURRENT_LINK = '/var/www/assignment8/current'
	}
	stages {
		stage('Checkout') {
			steps { checkout scm }
		}
		stage('Validate') {
			steps {
				echo 'Skipping lint for now (add htmlhint/stylelint later)'
			}
		}
		stage('Archive') {
			steps { archiveArtifacts artifacts: 'index.html,style.css', fingerprint: true }
		}
		stage('Deploy') {
			steps {
				sshagent (credentials: ['ec2_ssh_key']) {
					sh '''
					set -e
					TS=$(date +%s)
					TARGET="$RELEASE_DIR/$TS"
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
```

## Local Preview

Simply open `index.html` in your browser. No build step required.

## Next Ideas

- Add HTML/CSS linting.
- Add GitHub webhook triggering Jenkins.
- Introduce versioned artifacts (zip) + rollback step.
- Add CloudFront distribution in front of EC2 or migrate to S3 + CloudFront for static hosting.

---
Feel free to adapt and extend for future assignments.
