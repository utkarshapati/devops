pipeline {
    agent any

    tools {
        dependencyCheck 'DependencyCheck'
    }

    environment {
        NODE_OPTIONS = "--openssl-legacy-provider"
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
                    mkdir -p reports

                    gitleaks detect \
                        --no-git \
                        --source . \
                        --report-format sarif \
                        --report-path reports/gitleaks.sarif
                '''
            }
        }

        stage('Dependency Scan - OWASP') {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan .
                    --format HTML
                    --format XML
                    --out reports
                ''',
                odcInstallation: 'DependencyCheck'
            }
        }

        stage('Publish Dependency Report') {
            steps {
                dependencyCheckPublisher pattern: 'reports/dependency-check-report.xml'
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
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build React Application') {
            environment {
                CI = "false"
            }
            steps {
                sh 'npm run build'
            }
        }
    }

    post {

        always {
            archiveArtifacts artifacts: 'reports/*', fingerprint: true
            echo 'Pipeline execution finished.'
        }

        success {
            echo '✅ CI Pipeline completed successfully!'
        }

        failure {
            echo '❌ CI Pipeline failed.'
        }
    }
}
