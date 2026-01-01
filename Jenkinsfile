pipeline {
  agent any

  environment {
    DOCKERHUB_CREDS = 'DOCKERHUB_CREDS'
    GITHUB_PUSH_CREDS = 'GITHUB_PUSH'

    DOCKERHUB_USER = 'boogi0501'
    IMAGE_NAME = 'mantech-demo'
    IMAGE_FULL = "${DOCKERHUB_USER}/${IMAGE_NAME}"

    GITOPS_REPO = 'https://github.com/mantech-portfolio/mantech-gitops-manifests.git'
    GITOPS_BRANCH = 'main'
    DEPLOY_FILE = 'k8s/base/deployment.yaml'
  }

  stages {
    stage('Checkout App Repo') {
      steps {
        checkout scm
      }
    }

    stage('Build JAR') {
      steps {
        dir('app') {
          sh 'chmod +x gradlew'
          sh './gradlew clean test'
          sh './gradlew bootJar -x test'
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          env.TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        }
        dir('app') {
          sh "docker build -t ${IMAGE_FULL}:${TAG} ."
          sh "docker tag ${IMAGE_FULL}:${TAG} ${IMAGE_FULL}:latest"
        }
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_TOKEN')]) {
          sh 'echo $DH_TOKEN | docker login -u $DH_USER --password-stdin'
          sh "docker push ${IMAGE_FULL}:${TAG}"
          sh "docker push ${IMAGE_FULL}:latest"
        }
      }
    }

    stage('Update GitOps Manifest (image tag)') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${GITHUB_PUSH_CREDS}", usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          sh """
            rm -rf gitops
            git clone -b ${GITOPS_BRANCH} https://${GH_USER}:${GH_TOKEN}@${GITOPS_REPO.replace('https://','')} gitops
            cd gitops
            sed -i 's|image: ${IMAGE_FULL}:.*|image: ${IMAGE_FULL}:${TAG}|' ${DEPLOY_FILE}
            git add ${DEPLOY_FILE}
            git commit -m "chore: update image tag to ${TAG}" || echo "no changes"
            git push origin ${GITOPS_BRANCH}
          """
        }
      }
    }
  }
}

