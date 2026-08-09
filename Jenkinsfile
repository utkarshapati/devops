pipeline {

    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

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
                        --exit-code 0

                    echo "Gitleaks scan completed."
                '''
            }
        }

        stage('Dependency Scan - OWASP') {
            steps {
                sh 'mkdir -p reports'

                dependencyCheck(
                    odcInstallation: 'DependencyCheck',
                    nvdCredentialsId: 'nvd-api-key',
                    additionalArguments: '--noupdate --format XML --format HTML --out reports',
                    stopBuild: true
                )
            }
        }

        stage('Publish Dependency Report') {
            steps {

                dependencyCheckPublisher(
                    pattern: 'reports/dependency-check-report.xml',
                    skipNoReportFiles: false,
                    stopBuild: false
                )

                publishHTML(
                    target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports',
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'OWASP Dependency-Check Report'
                    ]
                )
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
                sh '''
                    echo "========== Installing Dependencies =========="
                    npm install
                '''
            }
        }

        stage('Build React Application') {
            steps {
                sh '''
                    echo "========== Building React Application =========="

                    CI=true npm run build
                '''
            }
        }
    }

    post {

        always {
            echo 'Pipeline execution finished.'
        }

        success {
            echo '✅ CI Pipeline completed successfully.'
        }

        failure {
            echo '❌ CI Pipeline failed.'
        }

        cleanup {
            echo 'Cleaning workspace...'
            cleanWs(
                deleteDirs: true,
                disableDeferredWipeout: true
            )
        }
    }
}
