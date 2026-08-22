pipeline {

    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        CI = 'false'

        IMAGE_NAME = 'prime-clone'
        IMAGE_TAG = 'v1'

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
                    echo "========== Environment =========="
                    echo "User:"
                    whoami

                    echo "Workspace:"
                    pwd

                    echo "========== Git =========="
                    git --version

                    echo "========== Node =========="
                    node --version

                    echo "========== NPM =========="
                    npm --version

                    echo "========== Docker =========="
                    docker --version

                    echo "========== Gitleaks =========="
                    gitleaks version

                    echo "========== Trivy =========="
                    trivy --version
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "========== Installing Dependencies =========="
                    npm install
                '''
            }
        }

        stage('Secret Scan - Gitleaks') {
            steps {
                sh '''
                    echo "========== Gitleaks Secret Scan =========="

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
                        echo "========== OWASP Dependency-Check =========="
                        echo "Using Dependency-Check:"
                        echo "${dependencyCheckHome}"

                        mkdir -p reports

                        ${dependencyCheckHome}/bin/dependency-check.sh \
                        --project "prime-clone" \
                        --scan . \
                        --format HTML \
                        --out reports \
                        --noupdate \
                        --disableAssembly \
                        --disableNodeAudit \
                        --disableRetireJS
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
                        echo "========== SonarQube Analysis =========="

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
                    echo "========== React Build =========="

                    CI=false npm run build
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "========== Docker Build =========="

                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Trivy Container Scan') {
            steps {
                sh '''
                    echo "========== Trivy Container Scan =========="

                    mkdir -p reports

                    trivy image \
                    --scanners vuln \
                    --severity HIGH,CRITICAL \
                    ${IMAGE_NAME}:${IMAGE_TAG}

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

        stage('Run Docker Container') {
            steps {
                sh '''
                    echo "========== Deploy Application =========="

                    docker rm -f prime-clone-app || true

                    docker run -d \
                    --name prime-clone-app \
                    -p 8081:80 \
                    ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "========== Running Container =========="

                    docker ps --filter "name=prime-clone-app"

                    echo "Application: http://localhost:8081"
                '''
            }
        }
    }

    post {

        success {
            echo '''
========================================
     DEVSECOPS PIPELINE SUCCESS
========================================
Application: http://localhost:8081
========================================
'''
        }

        failure {
            echo '''
========================================
     DEVSECOPS PIPELINE FAILED
========================================
Check the failed stage above.
========================================
'''
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
