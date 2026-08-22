pipeline {

    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        CI = 'false'

        IMAGE_NAME = 'prime-clone'
        IMAGE_TAG  = 'v1'

        SONAR_SCANNER = tool 'SonarQubeScanner'
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "========== VERIFY ENVIRONMENT =========="

                    echo "User:"
                    whoami

                    echo "Workspace:"
                    pwd

                    echo "Git:"
                    git --version

                    echo "Node:"
                    node --version

                    echo "NPM:"
                    npm --version

                    echo "Docker:"
                    docker --version

                    echo "Gitleaks:"
                    gitleaks version

                    echo "Trivy:"
                    trivy --version
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "========== INSTALL DEPENDENCIES =========="

                    npm install
                '''
            }
        }

        stage('Secret Scan - Gitleaks') {
            steps {
                sh '''
                    echo "========== GITLEAKS SECRET SCAN =========="

                    mkdir -p reports

                    gitleaks detect \
                        --source . \
                        --no-git \
                        --report-format sarif \
                        --report-path reports/gitleaks.sarif \
                        --exit-code 1
                '''
            }

            post {
                always {
                    archiveArtifacts(
                        artifacts: 'reports/gitleaks.sarif',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Dependency Scan - OWASP') {
            steps {
                script {

                    def dependencyCheckHome = tool 'DependencyCheck'

                    sh """
                        echo "========== OWASP DEPENDENCY-CHECK =========="

                        echo "Using Dependency-Check:"
                        echo "${dependencyCheckHome}"

                        mkdir -p reports

                        ${dependencyCheckHome}/bin/dependency-check.sh \
                            --project "prime-clone" \
                            --scan . \
                            --format HTML \
                            --out reports \
                            --noupdate \
                            --disableAssembly
                    """
                }
            }

            post {
                always {
                    archiveArtifacts(
                        artifacts: 'reports/dependency-check-report.html',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {

                    sh '''
                        echo "========== SONARQUBE ANALYSIS =========="

                        echo "Using scanner:"
                        echo "${SONAR_SCANNER}"

                        ${SONAR_SCANNER}/bin/sonar-scanner \
                            -Dsonar.projectKey=prime-clone \
                            -Dsonar.projectName=prime-clone \
                            -Dsonar.sources=src \
                            -Dsonar.exclusions=node_modules/**,build/** \
                            -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }

        stage('Build React Application') {
            steps {
                sh '''
                    echo "========== REACT BUILD =========="

                    CI=false npm run build

                    echo "React build completed."
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "========== DOCKER BUILD =========="

                    docker build \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} .

                    echo "========== DOCKER IMAGE =========="

                    docker images ${IMAGE_NAME}
                '''
            }
        }

        stage('Trivy Container Scan') {
            steps {
                sh '''
                    echo "========== TRIVY CONTAINER SCAN =========="

                    mkdir -p reports

                    echo "Scanning:"
                    echo "${IMAGE_NAME}:${IMAGE_TAG}"

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "Generating Trivy SARIF report..."

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --format sarif \
                        --output reports/trivy.sarif \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }

            post {
                always {
                    archiveArtifacts(
                        artifacts: 'reports/trivy.sarif',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Run Application Locally') {
            steps {
                sh '''
                    echo "========== RUN APPLICATION =========="

                    echo "Removing previous container if present..."

                    docker rm -f prime-clone-app || true

                    echo "Starting container..."

                    docker run -d \
                        --name prime-clone-app \
                        -p 8081:80 \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "========== CONTAINER STATUS =========="

                    docker ps \
                        --filter "name=prime-clone-app"

                    echo "Application URL:"
                    echo "http://localhost:8081"
                '''
            }
        }

        stage('Application Health Check') {
            steps {
                sh '''
                    echo "========== APPLICATION HEALTH CHECK =========="

                    sleep 5

                    curl -f http://localhost:8081

                    echo ""
                    echo "Application is running successfully!"
                '''
            }
        }
    }

    post {

        success {
            echo '''
==================================================
           DEVSECOPS PIPELINE SUCCESS
==================================================

Security:
  Gitleaks               PASS
  OWASP Dependency Check PASS
  SonarQube              PASS
  Trivy                  PASS

Build:
  React                  PASS
  Docker                 PASS

Deployment:
  Local Docker           PASS

Application:
  http://localhost:8081

==================================================
'''
        }

        failure {
            echo '''
==================================================
           DEVSECOPS PIPELINE FAILED
==================================================

Check the failed stage in Console Output.

==================================================
'''
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
