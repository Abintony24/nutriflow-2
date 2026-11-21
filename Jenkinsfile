pipeline {
    agent any

    options {
        timeout(time: 45, unit: 'MINUTES')
    }

    environment {
        BACKEND_IMAGE = 'abintony/nutriflow-backend'
        FRONTEND_IMAGE = 'abintony/nutriflow-frontend'
        GITHUB_REPO = 'https://github.com/Abintony24/nutriflow-2.git'
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
                    test -f k8s/deploy.yml
                    test -f k8s/service.yml

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

        stage('Trivy - Filesystem Scan') {
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
                      --build-arg VITE_API_URL="${VITE_API_URL}" \
                      -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                      -t ${FRONTEND_IMAGE}:latest \
                      ./frontend
                '''
            }
        }

        stage('Trivy - Docker Image Scan') {
            steps {
                sh '''
                    trivy image ${BACKEND_IMAGE}:${BUILD_NUMBER}
                    trivy image ${FRONTEND_IMAGE}:${BUILD_NUMBER}
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

        stage('Update Deployment Files') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'github-token',
                        variable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        git config user.name "abintony24"
                        git config user.email "abintony24@gmail.com"

                        sed -i \
                          "s|image: abintony/nutriflow-backend:.*|image: abintony/nutriflow-backend:${BUILD_NUMBER}|" \
                          k8s/deploy.yml

                        sed -i \
                          "s|image: abintony/nutriflow-frontend:.*|image: abintony/nutriflow-frontend:${BUILD_NUMBER}|" \
                          k8s/deploy.yml

                        git add k8s/deploy.yml

                        if git diff --cached --quiet; then
                            echo "No deployment file changes."
                            exit 0
                        fi

                        git commit -m "Update Nutriflow images to build ${BUILD_NUMBER}"

                        AUTH=$(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 -w 0)

                        git -c http.extraheader="AUTHORIZATION: basic ${AUTH}" \
                            push "$GITHUB_REPO" HEAD:main

                        unset AUTH
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Nutriflow CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Console Output for the failed stage.'
        }

        always {
            cleanWs()
        }
    }
}
