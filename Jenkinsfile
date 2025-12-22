pipeline {
  agent any

  environment {
    REGISTRY = "10.131.103.92:8090"
    PROJECT  = "kp_9"
    IMAGE_TAG = "${BUILD_NUMBER}"
    GIT_REPO = "https://github.com/ThanujaRatakonda/kp_9.git"
    DOCKER_USERNAME = "admin"
    DOCKER_PASSWORD = "Harbor12345"
    TRIVY_OUTPUT_JSON = "trivy-output.json"
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'ARGOCD_ONLY'],
      description: 'Run full pipeline, only frontend/backend, or just apply ArgoCD resources'
    )
    choice(
      name: 'ENV',
      choices: ['dev', 'qa', 'BOTH'],  // 🔥 NEW: Deploy BOTH at once!
      description: 'Choose environment(s): dev/qa/BOTH'
    )
    booleanParam(
      name: 'SKIP_SCANNING',
      defaultValue: false,
      description: 'Skip Trivy vulnerability scanning (for quick testing)'
    )
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }

    /* =========================
       SETUP ENVIRONMENTS (dev/qa/BOTH)
       ========================= */
    stage('Setup Environments') {
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              # Create namespace if needed
              kubectl get namespace ${ENV_NS} || kubectl create namespace ${ENV_NS}
              
              # Create Docker Registry Secret
              kubectl get secret regcred -n ${ENV_NS} || kubectl create secret docker-registry regcred -n ${ENV_NS} \
                --docker-server=${REGISTRY} \
                --docker-username=${DOCKER_USERNAME} \
                --docker-password=${DOCKER_PASSWORD}
              
              # Apply storage (PV/PVC)
              kubectl get pvc shared-pvc -n ${ENV_NS} || kubectl apply -f k8s/shared-pvc_${ENV_NS}.yaml -n ${ENV_NS}
              kubectl get pv shared-pv-${ENV_NS} || kubectl apply -f k8s/shared-pv_${ENV_NS}.yaml
              
              echo "✅ ${ENV_NS} environment ready!"
            """
          }
        }
      }
    }

    /* =========================
       FRONTEND: Build → Trivy → Push
       ========================= */
    stage('Build Frontend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t frontend:${IMAGE_TAG} ./frontend
          echo "✅ Frontend built: frontend:${IMAGE_TAG}"
        """
      }
    }

    stage('🛡️ Trivy Scan Frontend') {
      when { 
        expression { 
          params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] && 
          !params.SKIP_SCANNING
        } 
      }
      steps {
        sh """
          trivy image --severity CRITICAL,HIGH --format json -o ${TRIVY_OUTPUT_JSON} frontend:${IMAGE_TAG}
        """
        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true, allowEmptyArchive: true
        
        script {
          def props = readJSON file: "${TRIVY_OUTPUT_JSON}"
          def criticalHighCount = 0
          
          props.Results.each { result ->
            if (result.Vulnerabilities) {
              criticalHighCount += result.Vulnerabilities.count { 
                it.Severity in ['CRITICAL', 'HIGH'] 
              }
            }
          }
          
          echo "🔍 Frontend: ${criticalHighCount} CRITICAL/HIGH vulnerabilities"
          if (criticalHighCount > 0) {
            error "🚨 ${criticalHighCount} CRITICAL/HIGH vulnerabilities in frontend!"
          }
          echo "✅ Frontend scan PASSED!"
        }
      }
    }

    stage('Push Frontend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'harbor-creds',
          usernameVariable: 'USER',
          passwordVariable: 'PASS'
        )]) {
          sh """
            docker login ${REGISTRY} -u \$USER -p \$PASS
            docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
            docker rmi frontend:${IMAGE_TAG} || true
            echo "✅ Frontend pushed!"
          """
        }
      }
    }

    /* =========================
       BACKEND: Build → Trivy → Push
       ========================= */
    stage('Build Backend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t backend:${IMAGE_TAG} ./backend
          echo "✅ Backend built: backend:${IMAGE_TAG}"
        """
      }
    }

    stage('🛡️ Trivy Scan Backend') {
      when { 
        expression { 
          params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] && 
          !params.SKIP_SCANNING
        } 
      }
      steps {
        sh """
          trivy image --severity CRITICAL,HIGH --format json -o ${TRIVY_OUTPUT_JSON} backend:${IMAGE_TAG}
        """
        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true, allowEmptyArchive: true
        
        script {
          def props = readJSON file: "${TRIVY_OUTPUT_JSON}"
          def criticalHighCount = 0
          
          props.Results.each { result ->
            if (result.Vulnerabilities) {
              criticalHighCount += result.Vulnerabilities.count { 
                it.Severity in ['CRITICAL', 'HIGH'] 
              }
            }
          }
          
          echo "🔍 Backend: ${criticalHighCount} CRITICAL/HIGH vulnerabilities"
          if (criticalHighCount > 0) {
            error "🚨 ${criticalHighCount} CRITICAL/HIGH vulnerabilities in backend!"
          }
          echo "✅ Backend scan PASSED!"
        }
      }
    }

    stage('Push Backend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'harbor-creds',
          usernameVariable: 'USER',
          passwordVariable: 'PASS'
        )]) {
          sh """
            docker login ${REGISTRY} -u \$USER -p \$PASS
            docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
            docker rmi backend:${IMAGE_TAG} || true
            echo "✅ Backend pushed!"
          """
        }
      }
    }

    /* =========================
       UPDATE HELM VALUES FOR ALL ENVS
       ========================= */
    stage('Update Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              sed -i 's|tag:.*|tag: "${IMAGE_TAG}"|' frontend-hc/frontendvalues_${ENV_NS}.yaml
              sed -i 's|tag:.*|tag: "${IMAGE_TAG}"|' backend-hc/backendvalues_${ENV_NS}.yaml
              echo "✅ Updated ${ENV_NS} helm values to tag ${IMAGE_TAG}"
            """
          }
        }
      }
    }

    /* =========================
       COMMIT FOR ARGO CD (ALL ENVS)
       ========================= */
    stage('Commit & Push Helm Changes') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'GitHub',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          sh """
            git config user.name "thanuja"
            git config user.email "ratakondathanuja@gmail.com"
            git add frontend-hc/*.yaml backend-hc/*.yaml
            git commit -m "chore: update images ${IMAGE_TAG} for ${params.ENV}" || echo "No changes"
            git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_9.git master
            echo "✅ Committed changes for ${params.ENV}"
          """
        }
      }
    }

    /* =========================
       APPLY ARGOCD APPS (dev/qa/BOTH)
       ========================= */
    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY'] } }
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              # Apply ArgoCD apps for this env
              kubectl get application backend-${ENV_NS} -n argocd || kubectl apply -f argocd/backend-app_${ENV_NS}.yaml
              kubectl get application frontend-${ENV_NS} -n argocd || kubectl apply -f argocd/frontend-app_${ENV_NS}.yaml
              kubectl get application database-${ENV_NS} -n argocd || kubectl apply -f argocd/database-app_${ENV_NS}.yaml
              
              # Hard refresh ArgoCD
              kubectl annotate application backend-${ENV_NS} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application frontend-${ENV_NS} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application database-${ENV_NS} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              
              echo "✅ ArgoCD apps applied for ${ENV_NS}"
            """
          }
        }
      }
    }

    stage('✅ Verify Deployment') {
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              echo "=== ${ENV_NS} STATUS ==="
              kubectl get pods -n ${ENV_NS}
              kubectl get svc -n ${ENV_NS}
            """
          }
          sh "kubectl get applications -n argocd | grep -E '(dev|qa)'"
        }
      }
    }
  }

  post {
    always {
      sh 'docker logout ${REGISTRY} || true'
      sh 'docker image prune -f || true'
      archiveArtifacts artifacts: 'trivy-output.json', allowEmptyArchive: true
    }
  }
}
