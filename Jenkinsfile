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
                    whoami
                    pwd

                    echo "========== Git =========="
                    git --version

                    echo "========== Node =========="
                    node -v

                    echo "========== NPM =========="
                    npm -v

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
                sh 'npm install'
            }
        }

        stage('Secret Scan - Gitleaks') {
            steps {
                sh '''
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
                    archiveArtifacts artifacts: 'reports/gitleaks.sarif',
                    allowEmptyArchive: true
                }
            }
        }

        stage('Dependency Scan - OWASP') {
            steps {
                sh '''
                    mkdir -p reports

                    dependency-check \
                    --project "prime-clone" \
                    --scan . \
                    --format HTML \
                    --out reports \
                    --noupdate \
                    --disableAssembly \
                    --disableNodeAudit \
                    --disableRetireJS
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'reports/dependency-check-report.html',
                    allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        echo "========== SonarQube Analysis =========="

                        echo "Using scanner: ${SONAR_SCANNER}"

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
                    echo "========== Trivy Container Security Scan =========="

                    mkdir -p reports

                    trivy image \
                    --scanners vuln \
                    --severity HIGH,CRITICAL \
                    --format table \
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
                    archiveArtifacts artifacts: 'reports/trivy.sarif',
                    allowEmptyArchive: true
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                sh '''
                    echo "========== Deploy Container =========="

                    docker rm -f prime-clone-app 2>/dev/null || true

                    docker run -d \
                    --name prime-clone-app \
                    -p 8081:80 \
                    ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "========== Running Container =========="

                    docker ps --filter "name=prime-clone-app"
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo 'DevSecOps Pipeline Completed!'
            echo '======================================'
            echo 'Application: http://localhost:8081'
        }

        failure {
            echo '======================================'
            echo 'DevSecOps Pipeline Failed!'
            echo '======================================'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
