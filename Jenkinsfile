pipeline {
    agent any

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
        ARGOCD_SERVER = "your-argocd-server:443"
        ARGOCD_AUTH_TOKEN = credentials('argocd-token') // store token in Jenkins credentials
    }

    parameters {
        choice(
            name: 'SERVICE',
            choices: ['all', 'frontend', 'backend'],
            description: 'Choose which service to deploy'
        )
        string(name: 'FRONTEND_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for frontend')
        string(name: 'BACKEND_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for backend')
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp_9.git'
            }
        }

        stage('Build & Scan Frontend') {
            when { expression { params.SERVICE in ['all', 'frontend'] } }
            steps {
                sh "docker build -t frontend:${IMAGE_TAG} ./frontend"

                sh """
                    trivy image frontend:${IMAGE_TAG} \
                    --severity CRITICAL,HIGH \
                    --format json -o ${TRIVY_OUTPUT_JSON}
                """
                archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true

                script {
                    def vulns = sh(script: """
                        jq '[.Results[] |
                             (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                             (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                            ] | length' ${TRIVY_OUTPUT_JSON}
                    """, returnStdout: true).trim()

                    if (vulns.toInteger() > 0) {
                        error "CRITICAL/HIGH vulnerabilities found in frontend!"
                    }
                }
            }
        }

        stage('Push Frontend') {
            when { expression { params.SERVICE in ['all', 'frontend'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                        sh "docker tag frontend:${IMAGE_TAG} ${fullImage}"
                        sh "docker push ${fullImage}"
                        sh "docker rmi frontend:${IMAGE_TAG} || true"
                    }
                }
            }
        }

        stage('Build & Scan Backend') {
            when { expression { params.SERVICE in ['all', 'backend'] } }
            steps {
                sh "docker build -t backend:${IMAGE_TAG} ./backend"

                sh """
                    trivy image backend:${IMAGE_TAG} \
                    --severity CRITICAL,HIGH \
                    --format json -o ${TRIVY_OUTPUT_JSON}
                """
                archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true

                script {
                    def vulns = sh(script: """
                        jq '[.Results[] |
                             (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                             (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                            ] | length' ${TRIVY_OUTPUT_JSON}
                    """, returnStdout: true).trim()

                    if (vulns.toInteger() > 0) {
                        error "CRITICAL/HIGH vulnerabilities found in backend!"
                    }
                }
            }
        }

        stage('Push Backend') {
            when { expression { params.SERVICE in ['all', 'backend'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                        sh "docker tag backend:${IMAGE_TAG} ${fullImage}"
                        sh "docker push ${fullImage}"
                        sh "docker rmi backend:${IMAGE_TAG} || true"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f k8s/ -n dev"
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                script {
                    if (params.SERVICE in ['all', 'frontend']) {
                        sh "argocd app sync frontend --grpc-web --server ${env.ARGOCD_SERVER} --auth-token ${env.ARGOCD_AUTH_TOKEN}"
                    }
                    if (params.SERVICE in ['all', 'backend']) {
                        sh "argocd app sync backend --grpc-web --server ${env.ARGOCD_SERVER} --auth-token ${env.ARGOCD_AUTH_TOKEN}"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up Docker images..."
            sh "docker rmi frontend:${IMAGE_TAG} backend:${IMAGE_TAG} || true"
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check Trivy scan or deployment logs."
        }
    }
}
