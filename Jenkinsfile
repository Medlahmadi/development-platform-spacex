pipeline {
  agent any
  environment {
    DOCKERHUB_REG = "registry.hub.docker.com"
    DOCKERHUB_CRED = "dockerhub-creds"
    DOCKERHUB_NS   = "VotreNamespace"
    IMAGE_BACKEND  = "${DOCKERHUB_REG}/${DOCKERHUB_NS}/development-platform-spacex-backend"
    IMAGE_FRONTEND = "${DOCKERHUB_REG}/${DOCKERHUB_NS}/development-platform-spacex-frontend"
  }
  stages {
    stage('Checkout')   { steps { checkout scm } }
    stage('Install & Build') {
      steps {
        sh 'npm --prefix Frontend ci'
        sh './gradlew ngbuild'
        sh './gradlew StackRun'
      }
    }
    stage('Docker Build & Push') {
      steps {
        script {
          docker.withRegistry("https://${DOCKERHUB_REG}", DOCKERHUB_CRED) {
            def b = docker.build("${IMAGE_BACKEND}:$BUILD_NUMBER", "Backend")
            def f = docker.build("${IMAGE_FRONTEND}:$BUILD_NUMBER", "Frontend")
            b.push();  b.push('latest')
            f.push();  f.push('latest')
          }
        }
      }
    }
    stage('Deploy to Kubernetes') {
      steps {
        withKubeConfig([credentialsId: 'kubeconfig-id']) {
          sh '''
            kubectl apply -f k8s/postgres-pvc.yaml
            kubectl apply -f k8s/postgres-deployment.yaml
            kubectl wait --for=condition=ready pod -l app=postgres --timeout=120s
            kubectl apply -f k8s/backend-deployment.yaml
            kubectl apply -f k8s/backend-service.yaml
            kubectl apply -f k8s/frontend-deployment.yaml
            kubectl apply -f k8s/frontend-service.yaml
          '''
        }
      }
    }
  }
  post { always { cleanWs() } }
}
