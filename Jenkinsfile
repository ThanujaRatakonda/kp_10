pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
    }

    parameters {
        choice(
            name: 'SERVICE',
            choices: ['all', 'frontend', 'backend'],
            description: 'Which service to build and push'
        )
        string(name: 'ENV', defaultValue: 'dev', description: 'Namespace/Environment to deploy')
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp_9.git'
            }
        }

        stage('Build & Push Backend') {
            when { expression { params.SERVICE in ['all', 'backend'] } }
            steps {
                sh """
                docker build -t backend:${IMAGE_TAG} backend
                docker tag backend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                docker push ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                """
            }
        }

        stage('Build & Push Frontend') {
            when { expression { params.SERVICE in ['all', 'frontend'] } }
            steps {
                sh """
                docker build -t frontend:${IMAGE_TAG} frontend
                docker tag frontend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                docker push ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                """
            }
        }

        stage('Update Helm values') {
            steps {
                script {
                    if (params.SERVICE in ['all', 'backend']) {
                        sh "sed -i 's/tag:.*/tag: \"${IMAGE_TAG}\"/' backend-hc/backendvalues.yaml"
                    }
                    if (params.SERVICE in ['all', 'frontend']) {
                        sh "sed -i 's/tag:.*/tag: \"${IMAGE_TAG}\"/' frontend-hc/frontendvalues.yaml"
                    }
                }
            }
        }

        stage('Deploy K8s Manifests') {
            steps {
                script {
                    sh """
                    export ENV=${params.ENV}

                    # Apply k8s manifests to chosen namespace
                    for file in k8s/*; do
                        envsubst < \$file | kubectl apply -n \$ENV -f -
                    done
                    """
                }
            }
        }

        stage('Deploy ArgoCD Manifests') {
            steps {
                script {
                    sh """
                    # Apply argocd manifests in their own namespace
                    for file in argocd/*; do
                        kubectl apply -f \$file
                    done
                    """
                }
            }
        }

        stage('Commit & Push to Git') {
            steps {
                sh """
                git config user.name "thanuja"
                git config user.email "ratakondathanuja@gmail.com"
                git add .
                git commit -m "Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                git push origin master
                """
            }
        }
    }
}

