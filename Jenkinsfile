pipeline {
    agent any

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

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "========== Environment =========="
                    whoami
                    pwd

                    echo "========== PATH =========="
                    echo $PATH

                    echo "========== Git =========="
                    git --version

                    echo "========== Node =========="
                    which node || true
                    node -v || true

                    echo "========== NPM =========="
                    which npm || true
                    npm -v || true

                    echo "========== Docker =========="
                    docker --version || true
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                '''
            }
        }

        stage('Build React Application') {
            environment {
                NODE_OPTIONS = "--openssl-legacy-provider"
                CI = "false"
            }
            steps {
                sh '''
                    npm run build
                '''
            }
        }
    }

    post {

        always {
            archiveArtifacts artifacts: 'reports/*', fingerprint: true
            echo 'Pipeline execution finished.'
        }

        success {
            echo 'CI Pipeline completed successfully!'
        }

        failure {
            echo 'CI Pipeline failed.'
        }
    }
}
