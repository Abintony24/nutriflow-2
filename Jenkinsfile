pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        BACKEND_IMAGE = 'abintony/nutriflow-backend'
        FRONTEND_IMAGE = 'abintony/nutriflow-frontend'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    echo "Checking project files..."

                    test -f backend/Dockerfile
                    test -f frontend/Dockerfile
                    test -f docker-compose.yml

                    echo "Project file checks passed."
                '''
            }
        }

        stage('Semgrep - SAST') {
            steps {
                sh '''
                    semgrep scan --config p/security-audit .
                '''
            }
        }

        stage('Trivy - Security Scan') {
            steps {
                sh '''
                    trivy fs --scanners vuln,misconfig,secret .
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build \
                      -t ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                      -t ${BACKEND_IMAGE}:latest \
                      ./backend

                    docker build \
                      --build-arg VITE_API_URL="${VITE_API_URL:-}" \
                      -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                      -t ${FRONTEND_IMAGE}:latest \
                      ./frontend
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        'dockerhub-credentials'
                    ) {
                        sh '''
                            docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}
                            docker push ${BACKEND_IMAGE}:latest

                            docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                            docker push ${FRONTEND_IMAGE}:latest
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Nutriflow pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Console Output.'
        }

        always {
            cleanWs()
        }
    }
}
