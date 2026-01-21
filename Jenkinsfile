pipeline {
    agent any

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
                sh '''
                  set -euo pipefail

                  # ---- Update FRONTEND image: only inside container block "- name: frontend"
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

                  # ---- Update BACKEND image: only inside container block "- name: backend"
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

                  # ---- Guardrail: ensure initContainer image stays mysql:8.0
                  # This will fail the pipeline if someone accidentally overwrote it
                  grep -n "name: wait-for-mysql" -A3 backend-deploy.yaml | grep -q "image: mysql:8.0" \
                    || { echo "ERROR: initContainer wait-for-mysql image is NOT mysql:8.0"; exit 1; }

                  echo "✅ Updated images:"
                  grep -n "name: frontend" -A2 frontend-deploy.yaml | sed -n '1,3p'
                  grep -n "name: backend" -A2 backend-deploy.yaml | sed -n '1,6p'
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
                    sh """
                        set +e

                        git config --global user.name "jenkins-bot"
                        git config --global user.email "ci-bot@local"

                        git add mysql-deploy.yaml frontend-deploy.yaml backend-deploy.yaml

                        git commit -m "🔄 Update deployment image tags to ${IMAGE_TAG}"
                        COMMIT_STATUS=\$?

                        if [ \$COMMIT_STATUS -ne 0 ]; then
                          echo "No changes to commit (status=\$COMMIT_STATUS)"
                          exit 0
                        fi

                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/mira33ch/prototype-k3s.git ${BRANCH}
                    """
                }
            }
        }
    }

    post {
        success { echo 'CD pipeline executed successfully.' }
        failure { echo 'CD pipeline failed.' }
    }
}
