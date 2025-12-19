pipeline {
    agent any

    // Select environment dynamically
    parameters {
        choice(
            name: 'ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Target namespace / environment'
        )
    }

    environment {
        REGISTRY = "10.131.103.92:8090"
        PROJECT  = "kp_9"
        ENV      = "${params.ENV}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/ThanujaRatakonda/kp_7'
            }
        }

        stage('Calculate Release Version') {
            steps {
                script {
                    // Get last Git tag
                    def lastTag = sh(
                        script: "git tag --sort=-v:refname | head -n 1 || true",
                        returnStdout: true
                    ).trim()

                    // If no tag, start with v1.0.0, else increment patch version
                    if (!lastTag) {
                        env.RELEASE = "v1.0.0"
                    } else {
                        def v = lastTag.replace("v","").tokenize(".")
                        env.RELEASE = "v${v[0]}.${v[1]}.${v[2].toInteger()+1}"
                    }

                    echo "Release version: ${env.RELEASE}"
                }
            }
        }

        stage('Build & Push Backend') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'H_USER', passwordVariable: 'H_PASS')]) {
                    sh """
                        echo \$H_PASS | docker login ${REGISTRY} -u \$H_USER --password-stdin
                        docker build -t ${REGISTRY}/${PROJECT}/backend:${RELEASE} backend
                        docker push ${REGISTRY}/${PROJECT}/backend:${RELEASE}
                    """
                }
            }
        }

        stage('Build & Push Frontend') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'H_USER', passwordVariable: 'H_PASS')]) {
                    sh """
                        echo \$H_PASS | docker login ${REGISTRY} -u \$H_USER --password-stdin
                        docker build -t ${REGISTRY}/${PROJECT}/frontend:${RELEASE} frontend
                        docker push ${REGISTRY}/${PROJECT}/frontend:${RELEASE}
                    """
                }
            }
        }

        stage('Update Helm values') {
            steps {
                script {
                    // Update backend Helm values
                    sh "sed -i 's/tag:.*/tag: \"${RELEASE}\"/' backend-hc/backendvalues.yaml"
                    // Update frontend Helm values
                    sh "sed -i 's/tag:.*/tag: \"${RELEASE}\"/' frontend-hc/frontendvalues.yaml"
                }
            }
        }

        stage('Apply K8s & ArgoCD') {
            steps {
                sh """
                    export ENV=${ENV}
                    # Apply infra (namespace, storage, PVC)
                    for f in k8s/*.yaml; do
                        envsubst < \$f | kubectl apply -f -
                    done

                    # Apply ArgoCD applications
                    for f in argocd/*.yaml; do
                        envsubst < \$f | kubectl apply -f -
                    done
                """
            }
        }

        stage('Create Git Tag') {
            steps {
                sh """
                    git tag ${RELEASE}
                    git push origin ${RELEASE}
                """
            }
        }
    }
}
