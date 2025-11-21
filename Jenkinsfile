pipeline {
    agent any

    environment {
        BRANCH_NAME = "${env.GIT_BRANCH}".replaceAll("origin/", "")
        TIMESTAMP = "${new Date().format('yyyyMMddHHmmss')}"
        REPO_NAME = "react-test-project"
        IMAGE_PATH = "vikas0112/${REPO_NAME}"
        IMAGE_NAME = "${IMAGE_PATH}:${TIMESTAMP}"
    }

    stages {
        stage('Initialize Variables') {
            steps {
                script {
                    echo "Branch: ${BRANCH_NAME}"
                    echo "Repo Name: ${REPO_NAME}"
                    echo "Image Tag: ${TIMESTAMP}"
                }
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            agent {
                docker {
                    image 'node:18-alpine'
                }
            }
            steps {
                echo "Installing npm dependencies..."
                sh 'npm ci'
            }
        }

        stage('Build React App') {
            agent {
                docker {
                    image 'node:18-alpine'
                }
            }
            steps {
                echo "Running production build..."
                sh 'CI=false npm run build'
            }
        }

        stage('Lint Check') {
            agent {
                docker {
                    image 'node:18-alpine'
                }
            }
            steps {
                echo "Running ESLint check..."
                sh 'npm run lint || true'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image ${IMAGE_NAME}..."
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-registry-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        sh """
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push ${IMAGE_NAME}
                            docker logout
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    script {
                        sh """
                            export KUBECONFIG="$KUBECONFIG_FILE"
                            kubectl apply -f k8s/deployment.yaml -n default
                            kubectl apply -f k8s/service.yaml -n default
                            kubectl set image deployment/react-test-deployment react-test-container=${IMAGE_NAME} -n default --record
                            kubectl rollout status deployment/react-test-deployment -n default --timeout=180s
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
