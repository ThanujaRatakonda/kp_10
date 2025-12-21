pipeline {
    agent any

    parameters {
        choice(
            name: 'ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Target environment'
        )
    }

    environment {
        REGISTRY = "10.131.103.92:8090"
        PROJECT  = "kp_9"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                docker build -t $REGISTRY/$PROJECT/backend:$IMAGE_TAG backend
                docker build -t $REGISTRY/$PROJECT/frontend:$IMAGE_TAG frontend
                docker build -t $REGISTRY/$PROJECT/database:$IMAGE_TAG database
                '''
            }
        }

        stage('Login to Harbor') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-creds',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )
                ]) {
                    sh '''
                    echo "$HARBOR_PASS" | docker login $REGISTRY -u "$HARBOR_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                sh '''
                docker push $REGISTRY/$PROJECT/backend:$IMAGE_TAG
                docker push $REGISTRY/$PROJECT/frontend:$IMAGE_TAG
                docker push $REGISTRY/$PROJECT/database:$IMAGE_TAG
                '''
            }
        }

        stage('Update Helm Image Tags') {
            steps {
                sh '''
                echo "Updating Helm values to tag $IMAGE_TAG"

                sed -i "s|tag:.*|tag: \\"$IMAGE_TAG\\"|" backend-hc/backendvalues.yaml
                sed -i "s|tag:.*|tag: \\"$IMAGE_TAG\\"|" frontend-hc/frontendvalues.yaml
                sed -i "s|tag:.*|tag: \\"$IMAGE_TAG\\"|" database-hc/databasevalues.yaml

                echo "Backend:"
                grep tag backend-hc/backendvalues.yaml
                echo "Frontend:"
                grep tag frontend-hc/frontendvalues.yaml
                echo "Database:"
                grep tag database-hc/databasevalues.yaml
                '''
            }
        }

        stage('Commit Helm Changes') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'git-creds',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_PASS'
                    )
                ]) {
                    sh '''
                    git config user.email "ratakondathanuja@gmail.com"
                    git config user.name "thanuja"

                    git add .
                    git commit -m "Update image tags to $IMAGE_TAG for $ENV" || true

                    git push https://$GIT_USER:$GIT_PASS@github.com/ThanujaRatakonda/kp_9.git master
                    '''
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                echo "ArgoCD will auto-sync from Git (GitOps)"
            }
        }
    }

    post {
        success {
            echo "Deployment to $ENV completed successfully "
        }
        failure {
            echo "Deployment to $ENV failed "
        }
    }
}
