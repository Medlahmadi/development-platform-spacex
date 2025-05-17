pipeline {
  agent any

  environment {
    // Liaison des identifiants Docker Hub (username/password)
    DOCKER_CREDS = credentials('dockerhub-creds')
    // Préfixe des images
    IMAGE_BASE   = "${DOCKER_CREDS_USR}/development-platform-spacex"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Frontend') {
      steps {
        // Compile Angular
        sh './gradlew ngbuild'
      }
    }

    stage('Build, Test & Package') {
      steps {
        // Build backend + tests et création des images Docker
        sh './gradlew StackRun'
      }
    }

    stage('Publish Docker Images') {
      steps {
        // Connexion à Docker Hub et push des deux images
        sh '''
          echo "${DOCKER_CREDS_PSW}" | docker login -u "${DOCKER_CREDS_USR}" --password-stdin
          docker push ${IMAGE_BASE}-backend:latest
          docker push ${IMAGE_BASE}-frontend:latest
        '''
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        // Charge le kubeconfig stocké en tant que credential
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            export KUBECONFIG=${KUBECONFIG}

            # 1) Déployer Postgres et attendre son démarrage
            kubectl apply -f k8s/postgres.yaml
            kubectl rollout status deployment/postgres --timeout=120s

            # 2) Déployer le backend (avec initContainer qui attend Postgres)
            kubectl apply -f k8s/backend.yaml
            kubectl rollout status deployment/development-platform-spacex-backend --timeout=120s

            # 3) Déployer le frontend
            kubectl apply -f k8s/frontend.yaml
            kubectl rollout status deployment/development-platform-spacex-frontend --timeout=120s
          '''
        }
      }
    }
  }

  post {
    always {
      // Nettoyage facultatif : se déconnecter de Docker Hub
      sh 'docker logout'
    }
  }
}
