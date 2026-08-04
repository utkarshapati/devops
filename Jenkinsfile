pipeline {
    agent any

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
            }
            steps {
                sh '''
                    npm run build
                '''
            }
        }

    }

    post {

        success {
            echo '✅ CI Pipeline completed successfully!'
        }

        failure {
            echo '❌ CI Pipeline failed.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
