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
        
        sh label: 'Update images safely (bash)', script: '''
          #!/usr/bin/env bash
          set -euo pipefail

          echo "IMAGE_TAG=${IMAGE_TAG}"
          echo "Workspace: $(pwd)"
          ls -la

          # --- Update FRONTEND image (container name: frontend)
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

          # --- Update BACKEND image (container name: backend)
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

          # --- Guardrail: initContainer must stay mysql:8.0
          if ! grep -n "name: wait-for-mysql" -A3 backend-deploy.yaml | grep -q "image: mysql:8.0"; then
            echo "ERROR: initContainer wait-for-mysql image is NOT mysql:8.0"
            echo "Showing the initContainer block:"
            grep -n "initContainers:" -A12 backend-deploy.yaml || true
            exit 1
          fi

          echo "✅ Images after update:"
          echo "--- backend-deploy.yaml images ---"
          grep -n "image:" backend-deploy.yaml || true
          echo "--- frontend-deploy.yaml images ---"
          grep -n "image:" frontend-deploy.yaml || true
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
          sh label: 'Commit & Push (bash)', script: '''
            #!/usr/bin/env bash
            set -euo pipefail

            git config --global user.name "jenkins-bot"
            git config --global user.email "ci-bot@local"

            git add backend-deploy.yaml frontend-deploy.yaml mysql-deploy.yaml || true

            if git diff --cached --quiet; then
              echo "No changes detected, nothing to commit."
              exit 0
            fi

            git commit -m "🔄 Update deployment image tags to ${IMAGE_TAG}"

            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/mira33ch/prototype-k3s.git ${BRANCH}
          '''
        }
      }
    }
  }

  post {
    success { echo 'CD pipeline executed successfully.' }
    failure { echo 'CD pipeline failed.' }
    always  { echo "Done. IMAGE_TAG=${params.IMAGE_TAG}" }
  }
}
