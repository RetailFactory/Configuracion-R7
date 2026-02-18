pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    // ===== CAMBIAR =====
    APP_NAME        = "backend-r7"
    DOCKERHUB_USER  = "retailfactory"          // <- CAMBIAR
    IMAGE_REPO      = "${DOCKERHUB_USER}/${APP_NAME}"

    GITOPS_REPO_SSH = "git@github.com:RetailFactory/Configuracion-R7.git" // <- CAMBIAR
    GITOPS_BRANCH   = "main"

    GITOPS_OVERLAY_PATH = "apps/backend/overlays/dev" // <- Ajusta si tu estructura difiere
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install') {
      steps {
        sh 'npm ci'
      }
    }

    stage('Lint') {
      steps {
        // Si no tienes lint script, comenta esta línea
        sh 'npm run lint'
      }
    }

    stage('Test') {
      steps {
        // En Nest suele ser npm run test
        sh 'npm run test -- --ci'
      }
    }

    stage('Build App') {
      steps {
        sh 'npm run build'
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          env.GIT_SHA = sh(script: "git rev-parse --short=8 HEAD", returnStdout: true).trim()
          env.IMAGE_TAG = env.GIT_SHA
          env.IMAGE_FULL = "${env.IMAGE_REPO}:${env.IMAGE_TAG}"
        }
        sh '''
          docker build -t $IMAGE_FULL .
        '''
      }
    }

    stage('Push Docker Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',   // <- CAMBIAR si tu ID en Jenkins es otro
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_TOKEN'
        )]) {
          sh '''
            echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
            docker push $IMAGE_FULL
            docker logout
          '''
        }
      }
    }

    stage('Update GitOps (dev)') {
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: 'gitops-repo-ssh',   // <- CAMBIAR si tu ID en Jenkins es otro
          keyFileVariable: 'SSH_KEY'
        )]) {
          sh '''
            set -e
            rm -rf /tmp/gitops-r7-backend
            GIT_SSH_COMMAND="ssh -i $SSH_KEY -o StrictHostKeyChecking=no" \
              git clone -b $GITOPS_BRANCH $GITOPS_REPO_SSH /tmp/gitops-r7-backend

            cd /tmp/gitops-r7-backend/$GITOPS_OVERLAY_PATH

            sed -i "s#newTag: .*#newTag: $IMAGE_TAG#g" kustomization.yaml

            cd /tmp/gitops-r7-backend
            git config user.email "jenkins@r7.local"
            git config user.name "jenkins-r7"
            git add .
            git commit -m "backend(dev): image -> $IMAGE_TAG" || echo "No changes to commit"

            GIT_SSH_COMMAND="ssh -i $SSH_KEY -o StrictHostKeyChecking=no" \
              git push origin $GITOPS_BRANCH
          '''
        }
      }
    }
  }

  post {
    always {
      cleanWs()
    }
    success {
      echo "Backend pipeline completado. Imagen: ${env.IMAGE_FULL}"
    }
    failure {
      echo "Backend pipeline falló."
    }
  }
}
