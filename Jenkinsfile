pipeline {

    agent any

    tools {
        nodejs 'NodeJS'
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

                dependencyCheck(
                    odcInstallation: 'DependencyCheck',
                    additionalArguments: '--scan package-lock.json --noupdate --format HTML --format XML --out reports'
                )
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
                    sh '''
                        echo "========== React Build =========="
                        echo "CI=$CI"
                        npm run build
                    '''
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

        stage('Docker Build') {
            steps {
                sh '''
                    echo "========== Docker Build =========="

                    docker build -t prime-clone:latest .
                '''
            }
        }

        stage('Docker Image Check') {
            steps {
                sh '''
                    echo "========== Docker Image =========="

                    docker images prime-clone

                    docker inspect prime-clone:latest > /dev/null

                    echo "Docker image created successfully."
                '''
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
