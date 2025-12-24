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
        sh """
          kubectl get namespace ${params.ENV} >/dev/null 2>&1 || kubectl create namespace ${params.ENV}
          echo "✅ Namespace ${params.ENV} ready"
        """
      }
    }

    stage('Apply Storage') {
  steps {
    script {
      def ENV_NS = params.ENV
      def PV_FILE = "k8s/shared-pv_${ENV_NS}.yaml"
      def PVC_FILE = "k8s/shared-pvc_${ENV_NS}.yaml"
      
      sh """
        echo "🔧 FORCE CLEANING STUCK STORAGE for ${ENV_NS}..."
        
        # KILL STUCK FINALIZERS FIRST (CRITICAL!)
        kubectl patch pvc shared-pvc -n ${ENV_NS} -p '{"metadata":{"finalizers":null}}' --timeout=30s || true
        kubectl delete pvc shared-pvc -n ${ENV_NS} --grace-period=0 --force || true
        kubectl delete pv shared-pv-${ENV_NS} --grace-period=0 --force || true
        
        # Wait for cleanup
        sleep 5
        
        # CREATE HOST PATH
        mkdir -p /data/shared-${ENV_NS} || sudo mkdir -p /data/shared-${ENV_NS}
        chmod 777 /data/shared-${ENV_NS} || sudo chmod 777 /data/shared-${ENV_NS}
        
        echo "📦 Applying FRESH storage..."
        kubectl apply -f k8s/shared-storage-class.yaml
        kubectl apply -f ${PV_FILE}
        kubectl apply -f ${PVC_FILE}
        
        echo "⏳ SIMPLE WAIT FOR PVC BIND (no complex loops)..."
        for i in {1..30}; do
          sleep 6
          PHASE=\$(kubectl get pvc shared-pvc -n ${ENV_NS} -o jsonpath='{.status.phase}' 2>/dev/null || echo "ERROR")
          if [ "\$PHASE" = "Bound" ]; then 
            echo "✅ PVC BOUND IN ${i}s!"
            kubectl get pvc -n ${ENV_NS}
            exit 0
          fi
          echo "PVC [\$i/30]: \$PHASE"
        done
        
        echo "❌ PVC FAILED! DEBUG:"
        kubectl get pvc -n ${ENV_NS} -o yaml
        kubectl get pv | grep ${ENV_NS}
        kubectl describe pvc shared-pvc -n ${ENV_NS}
        exit 1
      """
    }
  }
}

    stage('Apply Docker Secret') {
      steps {
        sh """
          kubectl apply -f docker-registry-secret.yaml || true
          echo "✅ Docker secret applied to ${params.ENV}"
        """
      }
    }

    stage('Deploy Database') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'DATABASE_ONLY'] } }
      steps {
        sh """
          echo "Deploying Database for ${params.ENV}..."
          kubectl apply -f k8s/database-deployment.yaml -n ${params.ENV} || true
          
          for i in {1..24}; do
            READY=\$(kubectl get pod -l app=database -n ${params.ENV} -o jsonpath='{.items[0].status.containerStatuses[0].ready}' 2>/dev/null || echo "false")
            STATUS=\$(kubectl get pod -l app=database -n ${params.ENV} --no-headers -o custom-columns=STATUS:.status.phase 2>/dev/null || echo "Pending")
            echo "Database[\$i]: \$STATUS (ready: \$READY)"
            [ "\$READY" = "true" ] && [ "\$STATUS" = "Running" ] && echo "✅ Database READY!" && break
            sleep 5
          done
          kubectl get svc -l app=database -n ${params.ENV}
        """
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
            echo "\$HARBOR_PASS" | docker login ${REGISTRY} -u "\$HARBOR_USER" --password-stdin
            echo "✅ Docker login successful"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t frontend:${IMAGE_TAG} ./frontend
          docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker rmi frontend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "✅ Frontend: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t backend:${IMAGE_TAG} ./backend
          docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker rmi backend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "✅ Backend: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
        """
      }
    }

    stage('Update & Commit Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          sh """
            echo "Updating Helm values..."
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml
          """
          
          withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git config user.name "Thanuja"
              git config user.email "ratakondathanuja@gmail.com"
              git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml version.txt
              git commit -m "chore: images ${IMAGE_TAG} for ${params.ENV}" || echo "No changes"
              git push https://\$GIT_USER:\$GIT_TOKEN@github.com/ThanujaRatakonda/kp_10.git master
            """
          }
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        sh """
          echo "Applying ArgoCD for ${params.ENV}..."
          kubectl apply -f argocd/backend_${params.ENV}.yaml
          kubectl apply -f argocd/frontend_${params.ENV}.yaml
          kubectl apply -f argocd/database-app_${params.ENV}.yaml
          
          kubectl annotate application backend-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          kubectl annotate application frontend-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          kubectl annotate application database-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          
          echo "⏳ Waiting for ArgoCD sync..."
          sleep 10
        """
      }
    }

    stage('Verify Deployment') {
      steps {
        sh """
          echo "=== 🚀 FINAL STATUS ==="
          echo "Pods:"
          kubectl get pods -n ${params.ENV} -o wide
          echo -e "\\nServices:"
          kubectl get svc -n ${params.ENV}
          echo -e "\\nArgoCD Apps:"
          kubectl get applications -n argocd | grep ${params.ENV}
          echo -e "\\nPVC Status:"
          kubectl get pvc -n ${params.ENV}
          echo -e "\\nPV Status:"
          kubectl get pv | grep ${params.ENV}
        """
      }
    }
  }
  
  post {
    always {
      sh """
        echo "Pipeline completed for ENV=${params.ENV}, ACTION=${params.ACTION}"
        kubectl get pods -n ${params.ENV} || true
      """
    }
    success {
      echo "🎉 PIPELINE SUCCESSFUL!"
    }
    failure {
      echo "💥 PIPELINE FAILED!"
      sh "kubectl get events -n ${params.ENV} --sort-by='.lastTimestamp' | tail -10 || true"
    }
  }
}
