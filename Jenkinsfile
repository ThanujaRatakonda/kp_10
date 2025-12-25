pipeline {
  agent any

  environment {
    REGISTRY = "10.131.103.92:8090"
    PROJECT  = "kp_10"
    GIT_REPO = "https://github.com/ThanujaRatakonda/kp_10.git"
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'DATABASE_ONLY', 'ARGOCD_ONLY'],
      description: 'Run full pipeline or specific components'
    )
    choice(
      name: 'ENV',
      choices: ['dev', 'qa'],
      description: 'Choose environment (dev/qa)'
    )
    choice(
      name: 'VERSION_BUMP',
      choices: ['patch', 'minor', 'major'],
      description: 'Choose version bump: patch/minor/major'
    )
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }
    
    stage('Read & Update Version') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          def newVersion

          if (params.VERSION_BUMP == 'patch') {
            def patchFile = 'patch_counter.txt'
            def currentPatch = fileExists(patchFile) ? readFile(patchFile).trim().toInteger() : -1 
            newVersion = "v1.0.${currentPatch + 1}"
            writeFile file: patchFile, text: "${currentPatch + 1}"
          } else if (params.VERSION_BUMP == 'minor') {
            def minorFile = 'minor_counter.txt'
            def currentMinor = fileExists(minorFile) ? readFile(minorFile).trim().toInteger() : -1
            newVersion = "v1.1.${currentMinor + 1}"
            writeFile file: minorFile, text: "${currentMinor + 1}"
          } else if (params.VERSION_BUMP == 'major') {
            def majorFile = 'major_counter.txt'
            def currentMajor = fileExists(majorFile) ? readFile(majorFile).trim().toInteger() : -1
            newVersion = "v2.0.${currentMajor + 1}"
            writeFile file: majorFile, text: "${currentMajor + 1}"
          }

          env.IMAGE_TAG = newVersion
          writeFile file: 'version.txt', text: newVersion
          echo "${params.VERSION_BUMP} → ${newVersion}"
        }
      }
    }

    stage('Create Namespace') {
      steps {
        timeout(time: 2, unit: 'MINUTES') {
          script {
            sh """
              set -e
              echo "Ensuring namespace ${params.ENV} exists..."
              
              # Quick check with timeout
              if timeout 10 kubectl get namespace ${params.ENV} >/dev/null 2>&1; then
                echo "Namespace ${params.ENV} already exists"
              else
                echo "Creating namespace ${params.ENV}..."
                timeout 30 kubectl create namespace ${params.ENV}
                sleep 3
              fi
              
              # Final verification
              kubectl get namespace ${params.ENV}
              echo "Namespace ${params.ENV} ready ✅"
            """
          }
        }
      }
    }

    stage('Apply Storage') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          script {
            def ENV_NS = params.ENV
            def PV_FILE = "k8s/shared-pv_${ENV_NS}.yaml"
            def PVC_FILE = "k8s/shared-pvc_${ENV_NS}.yaml"

            sh """
              set -e
              echo "Applying storage for ${ENV_NS}..."
              kubectl apply -f k8s/shared-storage-class.yaml || true
              test -f ${PV_FILE} && kubectl apply -f ${PV_FILE}
              kubectl get pv shared-pv-${ENV_NS}
              test -f ${PVC_FILE} && kubectl apply -f ${PVC_FILE}
              
              for i in {1..30}; do
                PHASE=\$(kubectl get pvc shared-pvc -n ${ENV_NS} -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                [ "\$PHASE" = "Bound" ] && echo "PVC Bound! ✅" && break
                echo "PVC: \$PHASE (\$i/30)"
                sleep 5
              done
            """
          }
        }
      }
    }

    stage('Apply Docker Secret') {
      steps {
        timeout(time: 1, unit: 'MINUTES') {
          script {
            sh """
              set -e
              echo "Applying Docker registry secret to ${params.ENV}..."
              kubectl -n ${params.ENV} delete secret regcred --ignore-not-found=true || true
              kubectl apply -f docker-registry-secret.yaml -n ${params.ENV}
              kubectl get secret regcred -n ${params.ENV}
              echo "Docker secret applied ✅"
            """
          }
        }
      }
    }

    stage('Deploy Database') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          script {
            sh """
              set -e
              echo "Deploying Database for ${params.ENV}..."
              kubectl apply -f k8s/database-deployment.yaml -n ${params.ENV} || true
              for i in {1..48}; do
                READY=\$(kubectl get pod -l app=database -n ${params.ENV} -o jsonpath='{.items[0].status.containerStatuses[0].ready}' 2>/dev/null || echo "false")
                STATUS=\$(kubectl get pod -l app=database -n ${params.ENV} --no-headers -o custom-columns=STATUS:.status.phase 2>/dev/null || echo "Pending")
                echo "Database: \$STATUS (ready: \$READY) (\$i/48)"
                [ "\$READY" = "true" ] && [ "\$STATUS" = "Running" ] && echo "Database READY! ✅" && break
                sleep 5
              done
              kubectl get svc -l app=database -n ${params.ENV}
            """
          }
        }
      }
    }

    stage('Docker Login') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'harbor-creds',
            usernameVariable: 'HARBOR_USER',
            passwordVariable: 'HARBOR_PASS'
          )
        ]) {
          sh """
            echo "$HARBOR_PASS" | docker login ${REGISTRY} -u "$HARBOR_USER" --password-stdin
            echo "Docker login successful ✅"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          set -e
          echo "Building Frontend ${IMAGE_TAG}..."
          docker build -t frontend:${IMAGE_TAG} ./frontend
          docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker rmi frontend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "Frontend pushed: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} ✅"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          set -e
          echo "Building Backend ${IMAGE_TAG}..."
          docker build -t backend:${IMAGE_TAG} ./backend
          docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker rmi backend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "Backend pushed: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} ✅"
        """
      }
    }

    stage('Update & Commit Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          sh """
            set -e
            echo "Updating Helm values for ${params.ENV}..."
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml
            git diff frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml
          """

          withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git config user.name "Thanuja"
              git config user.email "ratakondathanuja@gmail.com"
              git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml version.txt
              git commit -m "chore: update images ${IMAGE_TAG} for ${params.ENV} [skip ci]" || echo "No changes to commit"
              git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_10.git master
              echo "Git push successful ✅"
            """
          }
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          script {
            sh """
              set -e
              echo "Applying ArgoCD apps for ${params.ENV}..."
              kubectl apply -f argocd/backend_${params.ENV}.yaml
              kubectl apply -f argocd/frontend_${params.ENV}.yaml
              kubectl apply -f argocd/database-app_${params.ENV}.yaml
              
              echo "Refreshing ArgoCD apps..."
              kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              echo "ArgoCD refresh triggered ✅"
            """
          }
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        timeout(time: 2, unit: 'MINUTES') {
          script {
            sh """
              echo "=== FINAL STATUS FOR ${params.ENV} ==="
              kubectl get pods -n ${params.ENV} -o wide
              kubectl get svc -n ${params.ENV}
              kubectl get pvc -n ${params.ENV}
              echo "--- ArgoCD Applications ---"
              kubectl get applications -n argocd | grep -E "(frontend|backend|database)"
              echo "Pipeline COMPLETE! 🎉"
            """
          }
        }
      }
    }
  }

  post {
    always {
      script {
        sh """
          echo "Pipeline finished at \$(date)"
          echo "Final namespace status:"
          kubectl get ns ${params.ENV} || true
        """
      }
    }
    success {
      echo "FULL_PIPELINE SUCCESS ✅ ${params.ENV} environment deployed with ${IMAGE_TAG}"
    }
    failure {
      echo "Pipeline FAILED ❌ Check logs above"
    }
  }
}
