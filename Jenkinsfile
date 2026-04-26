pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    K8S_NAMESPACE = 'mini-project'
    BACKEND_IMAGE = 'task-manager-backend'
    FRONTEND_IMAGE = 'task-manager-frontend'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Images') {
      steps {
        script {
          env.IMAGE_TAG = "build-${env.BUILD_NUMBER}"
        }
        sh "docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} backend"
        sh "docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} frontend"
      }
    }

    stage('Load Images Into Minikube') {
      steps {
        sh "minikube image load ${BACKEND_IMAGE}:${IMAGE_TAG}"
        sh "minikube image load ${FRONTEND_IMAGE}:${IMAGE_TAG}"
      }
    }

    stage('Deploy') {
      environment {
        SUPABASE_URL = credentials('supabase-url')
        SUPABASE_SERVICE_ROLE_KEY = credentials('supabase-service-role-key')
        JWT_SECRET = credentials('jwt-secret')
      }
      steps {
        sh 'kubectl apply -f k8s/namespace.yaml'
        sh 'kubectl apply -f k8s/configmap.yaml'
        sh '''
          kubectl -n ${K8S_NAMESPACE} create secret generic task-manager-secrets \
            --from-literal=SUPABASE_URL="${SUPABASE_URL}" \
            --from-literal=SUPABASE_SERVICE_ROLE_KEY="${SUPABASE_SERVICE_ROLE_KEY}" \
            --from-literal=JWT_SECRET="${JWT_SECRET}" \
            --dry-run=client -o yaml | kubectl apply -f -
        '''
        sh 'kubectl apply -f k8s/backend.yaml'
        sh 'kubectl apply -f k8s/frontend.yaml'
        sh "kubectl -n ${K8S_NAMESPACE} set image deployment/backend backend=${BACKEND_IMAGE}:${IMAGE_TAG}"
        sh "kubectl -n ${K8S_NAMESPACE} set image deployment/frontend frontend=${FRONTEND_IMAGE}:${IMAGE_TAG}"
        sh "kubectl -n ${K8S_NAMESPACE} rollout status deployment/backend --timeout=180s"
        sh "kubectl -n ${K8S_NAMESPACE} rollout status deployment/frontend --timeout=180s"
      }
    }

    stage('Verify') {
      steps {
        sh 'kubectl -n ${K8S_NAMESPACE} get pods'
        sh 'kubectl -n ${K8S_NAMESPACE} get svc'
      }
    }
  }

  post {
    always {
      sh 'kubectl -n ${K8S_NAMESPACE} get all || true'
    }
  }
}
