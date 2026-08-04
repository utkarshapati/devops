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

        stage('Verify Environment') {
            steps {
                sh 'echo "===== Environment ====="'
                sh 'git --version'
                sh 'node --version'
                sh 'npm --version'
                sh 'docker --version'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build React Application') {
            steps {
                sh 'npm run build'
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
    }
}
