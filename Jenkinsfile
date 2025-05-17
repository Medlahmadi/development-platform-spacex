pipeline {
  agent any

  environment {
    DOCKER_CREDS   = credentials('dockerhub-creds')
    IMAGE_BASE     = "${DOCKER_CREDS_USR}/development-platform-spacex"
  }

  tools {
    jdk 'JDK17'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Frontend') {
      steps {
        sh './gradlew ngbuild'      // compilation Angular → dist/
      }
    }

    stage('Build, Test & Image') {
      steps {
        sh './gradlew StackRun'     // build backend, tests, création des images Docker
      }
    }

    stage('Push to Docker Hub') {
      steps {
        sh """
          echo '${DOCKER_CREDS_PSW}' | docker login -u '${DOCKER_CREDS_USR}' --password-stdin
          docker push ${IMAGE_BASE}-backend:latest
          docker push ${IMAGE_BASE}-frontend:latest
        """
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$KUBECONFIG_FILE
            # 1) Déployer Postgres
            kubectl apply -f k8s/postgres.yaml
            kubectl rollout status deployment/postgres --timeout=2m

            # 2) Déployer backend (avec initContainer pour attendre Postgres)
            kubectl apply -f k8s/backend.yaml
            kubectl rollout status deployment/development-platform-spacex-backend --timeout=2m

            # 3) Déployer frontend
            kubectl apply -f k8s/frontend.yaml
            kubectl rollout status deployment/development-platform-spacex-frontend --timeout=2m
          '''
        }
      }
    }
  }
}
