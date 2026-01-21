pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag to deploy')
  }

  environment {
    GIT_REPO = 'https://github.com/mira33ch/prototype-k3s.git'
    BRANCH   = 'main'
  }

  stages {
    stage('Clone GitOps Repo') {
      steps {
        git branch: "${BRANCH}", credentialsId: 'github', url: "${GIT_REPO}"
      }
    }

    stage('Update Deployment Files') {
      steps {
        sh label: 'Update images safely', script: '''
          #!/usr/bin/env bash
          set -euo pipefail

          echo "IMAGE_TAG=${IMAGE_TAG}"

          # Update frontend image (container name: frontend)
          awk -v tag="${IMAGE_TAG}" '
            BEGIN { in_frontend=0 }
            /^[[:space:]]*-[[:space:]]*name:[[:space:]]*frontend[[:space:]]*$/ { in_frontend=1 }
            in_frontend && /^[[:space:]]*image:[[:space:]]*/ {
              sub(/image:.*/, "image: mariem360/portfolio-app-cicd-pipeline-frontend:" tag)
              in_frontend=0
            }
            { print }
          ' frontend-deploy.yaml > frontend-deploy.yaml.tmp
          mv frontend-deploy.yaml.tmp frontend-deploy.yaml

          # Update backend image (container name: backend)
          awk -v tag="${IMAGE_TAG}" '
            BEGIN { in_backend=0 }
            /^[[:space:]]*-[[:space:]]*name:[[:space:]]*backend[[:space:]]*$/ { in_backend=1 }
            in_backend && /^[[:space:]]*image:[[:space:]]*/ {
              sub(/image:.*/, "image: mariem360/portfolio-app-cicd-pipeline-backend:" tag)
              in_backend=0
            }
            { print }
          ' backend-deploy.yaml > backend-deploy.yaml.tmp
          mv backend-deploy.yaml.tmp backend-deploy.yaml

          # Guardrail: initContainer must stay mysql:8.0
          grep -n "name: wait-for-mysql" -A3 backend-deploy.yaml | grep -q "image: mysql:8.0" \
            || { echo "ERROR: initContainer wait-for-mysql image is NOT mysql:8.0"; exit 1; }

          echo "✅ Images after update:"
          echo "--- backend-deploy.yaml images ---"
          grep -n "image:" backend-deploy.yaml
          echo "--- frontend-deploy.yaml images ---"
          grep -n "image:" frontend-deploy.yaml
        '''
      }
    }

    stage('Commit & Push Changes') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github',
          usernameVariable: 'GIT_USERNAME',
          passwordVariable: 'GIT_PASSWORD'
        )]) {
          sh '''
            #!/usr/bin/env bash
            set -e

            git config --global user.name "jenkins-bot"
            git config --global user.email "ci-bot@local"

            git add backend-deploy.yaml frontend-deploy.yaml mysql-deploy.yaml || true

            git commit -m "🔄 Update deployment image tags to ${IMAGE_TAG}" || {
              echo "No changes to commit"
              exit 0
            }

            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/mira33ch/prototype-k3s.git ${BRANCH}
          '''
        }
      }
    }
  }

  post {
    success { echo 'CD pipeline executed successfully.' }
    failure { echo 'CD pipeline failed.' }
  }
}
