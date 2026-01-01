pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    skipDefaultCheckout(true)
  }

  environment {
    DOCKERHUB_CREDS   = 'DOCKERHUB_CREDS'
    GITHUB_PUSH_CREDS = 'GITHUB_PUSH'

    IMAGE_FULL = 'boogi0501/mantech-demo'

    GITOPS_REPO   = 'https://github.com/mantech-portfolio/mantech-gitops-manifests.git'
    GITOPS_BRANCH = 'main'
    DEPLOY_FILE   = 'k8s/base/deployment.yaml'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build JAR') {
      steps {
        dir('app') {
          sh '''
            set -euo pipefail
            chmod +x gradlew
            ./gradlew --no-daemon clean test bootJar
          '''
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          env.TAG = (env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : sh(script: "git rev-parse --short HEAD", returnStdout: true).trim())
        }
        dir('app') {
          sh '''
            set -euo pipefail
            docker build -t ${IMAGE_FULL}:${TAG} .
            docker tag ${IMAGE_FULL}:${TAG} ${IMAGE_FULL}:latest
          '''
        }
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDS, usernameVariable: 'DH_USER', passwordVariable: 'DH_TOKEN')]) {
          sh '''
            set -euo pipefail

            export DOCKER_CONFIG="$WORKSPACE/.docker"
            mkdir -p "$DOCKER_CONFIG"

            echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
            docker push ${IMAGE_FULL}:${TAG}
            docker push ${IMAGE_FULL}:latest
            docker logout

            rm -rf "$DOCKER_CONFIG"
          '''
        }
      }
    }

    stage('Update GitOps Manifest (image tag)') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.GITHUB_PUSH_CREDS, usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          sh '''
            set -euo pipefail
            rm -rf gitops

            export GIT_TERMINAL_PROMPT=0
            ASKPASS="$(mktemp)"
            cat > "$ASKPASS" <<'EOS'
#!/bin/sh
case "$1" in
  Username*) echo "$GH_USER" ;;
  Password*) echo "$GH_TOKEN" ;;
esac
EOS
            chmod +x "$ASKPASS"
            export GIT_ASKPASS="$ASKPASS"

            git clone -b "$GITOPS_BRANCH" "$GITOPS_REPO" gitops
            cd gitops

            git config user.name  "jenkins-bot"
            git config user.email "jenkins@local"

            # DEPLOY_FILE 안의 image 라인이 `${IMAGE_FULL}:...` 형태라고 가정
            sed -i "s|image: ${IMAGE_FULL}:.*|image: ${IMAGE_FULL}:${TAG}|" "${DEPLOY_FILE}"

            git add "${DEPLOY_FILE}"
            if git diff --cached --quiet; then
              echo "no changes"
              exit 0
            fi

            git commit -m "chore: update image tag to ${TAG}"
            git push origin "$GITOPS_BRANCH"

            rm -f "$ASKPASS"
          '''
        }
      }
    }
  }

  post {
    always {
      deleteDir()
    }
  }
}

