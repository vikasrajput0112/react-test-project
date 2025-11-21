pipeline {
  agent any
  environment {
    IMAGE_NAME = "vikas0112/react-test-project"
    TAG = "${env.BUILD_ID}-${env.GIT_COMMIT?.substring(0,7) ?: 'local'}"
    DOCKER_CREDS = "docker-registry-creds"
    KUBE_CRED = "kubeconfig"
    NAMESPACE = "default"
    DEPLOYMENT = "react-test-deployment"
    CONTAINER_NAME = "react-test-container"
  }
  options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install & Build React App') {
      steps {
        script {
          // Use bat commands on Windows Jenkins agent
          bat 'npm install'
          bat 'npm run build'
        }
      }
    }

    stage('Build Image & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          script {
            bat """
              echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
              docker build -t %IMAGE_NAME%:%TAG% .
              docker push %IMAGE_NAME%:%TAG%
              docker logout
            """
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([file(credentialsId: env.KUBE_CRED, variable: 'KUBECONFIG_FILE')]) {
          script {
            bat """
              set KUBECONFIG=%KUBECONFIG_FILE%
              kubectl apply -f k8s/deployment.yaml -n %NAMESPACE%
              kubectl apply -f k8s/service.yaml -n %NAMESPACE%
              kubectl set image deployment/%DEPLOYMENT% %CONTAINER_NAME%=%IMAGE_NAME%:%TAG% -n %NAMESPACE% --record
              kubectl rollout status deployment/%DEPLOYMENT% -n %NAMESPACE% --timeout=180s
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "Deployed image: ${env.IMAGE_NAME}:${env.TAG}"
    }
    failure {
      echo "Pipeline failed. Check logs."
    }
    always {
      bat 'echo Finished pipeline || exit /b 0'
    }
  }
}
