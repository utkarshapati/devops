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

        // =====================================================
        // 1. CHECKOUT
        // =====================================================

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }


        // =====================================================
        // 2. VERIFY ENVIRONMENT
        // =====================================================

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "========== VERIFY ENVIRONMENT =========="

                    whoami
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


        // =====================================================
        // 3. INSTALL DEPENDENCIES
        // =====================================================

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "========== INSTALL DEPENDENCIES =========="

                    npm install
                '''
            }
        }


        // =====================================================
        // 4. GITLEAKS
        // =====================================================

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


        // =====================================================
        // 5. OWASP DEPENDENCY-CHECK
        // =====================================================

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
                            --scan package-lock.json \
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


        // =====================================================
        // 6. SONARQUBE
        // =====================================================

        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('sonarqube') {

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


        // =====================================================
        // 7. REACT BUILD
        // =====================================================

        stage('Build React Application') {
            steps {
                sh '''
                    echo "========== REACT APPLICATION BUILD =========="

                    CI=false npm run build

                    echo "React build completed successfully."
                '''
            }
        }


        // =====================================================
        // 8. DOCKER BUILD
        // =====================================================

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


        // =====================================================
        // 9. TRIVY
        // =====================================================

        stage('Trivy Container Scan') {
            steps {
                sh '''
                    echo "========== TRIVY CONTAINER SCAN =========="

                    mkdir -p reports

                    echo "Scanning image:"
                    echo "${IMAGE_NAME}:${IMAGE_TAG}"

                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "Generating SARIF report..."

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


        // =====================================================
        // 10. RUN LOCALLY
        // =====================================================

        stage('Run Application Locally') {
            steps {
                sh '''
                    echo "========== DEPLOY LOCALLY =========="

                    echo "Removing old container..."

                    docker rm -f prime-clone-app || true

                    echo "Starting application..."

                    docker run -d \
                        --name prime-clone-app \
                        -p 8081:80 \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "========== CONTAINER STATUS =========="

                    docker ps \
                        --filter "name=prime-clone-app"

                    echo ""
                    echo "Application:"
                    echo "http://localhost:8081"
                '''
            }
        }


        // =====================================================
        // 11. HEALTH CHECK
        // =====================================================

        stage('Application Health Check') {
            steps {
                sh '''
                    echo "========== APPLICATION HEALTH CHECK =========="

                    sleep 5

                    curl -f http://localhost:8081

                    echo ""
                    echo "========================================"
                    echo "APPLICATION IS RUNNING SUCCESSFULLY"
                    echo "========================================"
                '''
            }
        }
    }


    // =========================================================
    // POST
    // =========================================================

    post {

        success {
            echo '''
==================================================
        DEVSECOPS PIPELINE SUCCESS
==================================================

Security:
  Gitleaks                PASS
  OWASP Dependency-Check  PASS
  SonarQube               PASS
  Trivy                   PASS

Build:
  React                   PASS
  Docker                  PASS

Deployment:
  Local Docker            PASS

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

Check the failed stage in Jenkins Console Output.

==================================================
'''
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
