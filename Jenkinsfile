pipeline {

    agent any

    tools {
        nodejs 'NodeJS'
        dependencyCheck 'DependencyCheck'
    }

    environment {
        NVD_API_KEY = credentials('nvd-api-key')
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
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
        }

        stage('Dependency Scan - OWASP') {
            steps {
                sh 'mkdir -p reports'

                withCredentials([
                    string(
                        credentialsId: 'nvd-api-key',
                        variable: 'NVD_API_KEY'
                    )
                ]) {

                    dependencyCheck(
                        odcInstallation: 'DependencyCheck',
                        additionalArguments: "--scan . --format HTML --format XML --out reports --nvdApiKey ${NVD_API_KEY}"
                    )
                }
            }
        }

        stage('Publish Dependency Report') {
            steps {
                publishHTML([
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'reports',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency-Check Report',
                    reportTitles: 'Dependency Vulnerability Report'
                ])
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
                    which node
                    node -v

                    echo "========== NPM =========="
                    which npm
                    npm -v

                    echo "========== Docker =========="
                    docker --version
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build React Application') {
            steps {
                withEnv(['CI=false']) {
                    sh 'npm run build'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {

                    sh '''
                        echo "========== SonarQube Analysis =========="

                        sonar-scanner \
                          -Dsonar.projectKey=prime-clone \
                          -Dsonar.projectName=prime-clone \
                          -Dsonar.sources=src \
                          -Dsonar.exclusions=node_modules/**,build/** \
                          -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
        }

        success {
            echo '✅ CI/CD Pipeline completed successfully.'
        }

        failure {
            echo '❌ CI/CD Pipeline failed.'
        }
    }
}
