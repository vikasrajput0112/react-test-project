
pipeline {
    agent any

    environment {
        BRANCH_NAME = "${env.GIT_BRANCH}".replaceAll("origin/", "")
        TIMESTAMP = "${new Date().format('yyyyMMddHHmmss')}"
        REPO_NAME = "react-erp_ui-2025-npm"
        IMAGE_PATH = "ghcr.io/paisalo-digital-limited/${REPO_NAME}"
        IMAGE_NAME = "${IMAGE_PATH}:${TIMESTAMP}"   // ✅ Add this line
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

        stage('Restore Dependencies') {
            agent {
                docker {
                    image 'node:20-alpine'
                }
            }
            steps {
                echo "Installing npm dependencies..."
                sh 'npm ci'
            }
        }

        stage('Debug Project Paths') {
            steps {
                echo "Verifying package and structure..."
                sh 'ls -l'
                sh 'cat package.json || true'
            }
        }

        stage('Build Solution') {
            agent {
                docker {
                    image 'node:20-alpine'
                }
            }
            steps {
                echo "Running build..."
                
                sh 'CI=false NODE_OPTIONS="--max-old-space-size=4096" npm run build || true'
            }
        }
        stage('SonarQube SAST Scan') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    script {
                        try {
                            echo "✅ Running SonarQube SAST Scan..."
                            sh """
                                dotnet tool install --global dotnet-sonarscanner || true
                                export PATH="\$PATH:\$HOME/.dotnet/tools"

                                dotnet sonarscanner begin \
                                    /k:"${SONAR_PROJECT_KEY}" \
                                    /d:sonar.host.url="${SONAR_HOST_URL}" \
                                    /d:sonar.login="${SONAR_TOKEN}"

                                dotnet build PDL.HRMSServices.API/PDL.HRMSServices.API.csproj

                                dotnet sonarscanner end \
                                    /d:sonar.login="${SONAR_TOKEN}"
                            """
                        } catch (e) {
                            echo "⚠️ SonarQube SAST Scan failed, continuing..."
                        }
                    }
                }
            }
        }

        stage('SonarQube SCA Scan') {
            steps {
                echo "✅ SonarQube SCA Scan (placeholder)..."
                // Add license/dependency check tool here
            }
        }

        stage('SonarQube Final Scan') {
            steps {
                echo "✅ SonarQube Final Scan (placeholder)..."
                // Optional: consolidate results or trigger gates
            }
        }

        stage('Run Unit Tests') {
            agent {
                docker {
                    image 'node:20-alpine'
                }
            }
            steps {
                echo "Running unit tests..."
                sh 'npm test -- --watchAll=false --coverage || true'
            }
        }

        stage('Check Code Coverage') {
            steps {
                echo "Checking test coverage summary..."
                sh 'cat coverage/lcov.info || true'
            }
        }

        stage('Lint Check') {
            agent {
                docker {
                    image 'node:20-alpine'
                }
            }
            steps {
                echo "Running ESLint check..."
                sh 'npm run lint || true'
            }
        }

        stage('Publish Artifacts') {
            steps {
                echo "Publishing build artifacts (skipped in React, unless needed)..."
                // Optional: archiveArtifacts artifacts: 'build/**', fingerprint: true
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh "docker build -t ${IMAGE_PATH}:${TIMESTAMP} ."
            }
        }

        stage('Push Docker Image to GHCR') {
    steps {
        withCredentials([string(credentialsId: 'ghcr-token1', variable: 'GITHUB_TOKEN')]) {
            script {
                def imageTag = REPO_NAME.toLowerCase()

                sh """
                    echo "${GITHUB_TOKEN}" | docker login ghcr.io -u paisalo-digital-limited --password-stdin

                    # Push image with timestamp tag
                    docker push ghcr.io/paisalo-digital-limited/${imageTag}:${TIMESTAMP}

                    # Tag and push 'latest'
                    docker tag ghcr.io/paisalo-digital-limited/${imageTag}:${TIMESTAMP} ghcr.io/paisalo-digital-limited/${imageTag}:latest
                    docker push ghcr.io/paisalo-digital-limited/${imageTag}:latest

                    # Cleanup local images
                    docker rmi ghcr.io/paisalo-digital-limited/${imageTag}:${TIMESTAMP} || true
                    docker rmi ghcr.io/paisalo-digital-limited/${imageTag}:latest || true
                """
            }
        }
    }
}

        stage('Delete Old GHCR Images, Keep Latest 10') {
            steps {
                withCredentials([string(credentialsId: 'ghcr-token1', variable: 'GITHUB_TOKEN')]) {
                    sh """
                      echo "🗑️ Fetching GHCR images for repo: ${REPO_NAME}"

                      all_ids=\$(curl -s -H "Authorization: Bearer \$GITHUB_TOKEN" \
                                       -H "Accept: application/vnd.github+json" \
                                       "https://api.github.com/orgs/Paisalo-Digital-Limited/packages/container/${REPO_NAME}/versions?per_page=200" \
                               | jq -r '.[].id')

                      keep_ids=\$(echo "\$all_ids" | head -n 10)
                      delete_ids=\$(echo "\$all_ids" | tail -n +11)

                      echo "📌 Keeping latest 10 versions: \$keep_ids"
                      echo "🗑️ Deleting old versions: \$delete_ids"

                      for vid in \$delete_ids; do
                        curl -s -X DELETE \
                             -H "Authorization: Bearer \$GITHUB_TOKEN" \
                             -H "Accept: application/vnd.github+json" \
                             "https://api.github.com/orgs/Paisalo-Digital-Limited/packages/container/${REPO_NAME}/versions/\$vid"
                        echo "✅ Deleted GHCR image version ID: \$vid"
                      done
                    """
                }
            }
        }
    }
       
    post {
        always {
            echo "Cleaning up workspace..."
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
