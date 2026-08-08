pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
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
                    echo "========== Gitleaks Secret Scan =========="

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
                withCredentials([
                    string(
                        credentialsId: 'nvd-api-key',
                        variable: 'NVD_API_KEY'
                    )
                ]) {
                    sh 'mkdir -p reports'

                    dependencyCheck(
                        odcInstallation: 'DependencyCheck',
                        additionalArguments: """
                            --scan .
                            --format HTML
                            --format XML
                            --out reports
                            --nvdApiKey ${NVD_API_KEY}
                        """
                    )
                }
            }
        }

        stage('Publish Dependency Report') {
            steps {
                dependencyCheckPublisher(
                    pattern: 'reports/dependency-check-report.xml'
                )
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
                sh '''
                    echo "========== Installing Dependencies =========="
                    npm install
                '''
            }
        }

        stage('Build React Application') {
            environment {
                CI = "false"
            }

            steps {
                sh '''
                    echo "========== Building React Application =========="
                    npm run build
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline execution finished."

            archiveArtifacts(
                artifacts: 'reports/*',
                allowEmptyArchive: true,
                fingerprint: true
            )
        }

        success {
            echo "✅ CI Pipeline completed successfully!"
        }

        failure {
            echo "❌ CI Pipeline failed."
        }
    }
}
