pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  environment {
    KUSTOMIZE_DIR = 'k8s/overlays/prod'
    KUBE_NS       = 'registry'
    REGISTRY_SVC  = 'registry'        // имя Service с type=NodePort
    TG_CHAT       = '180424264'       // можно тоже унести в креды
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Kubectl version') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            kubectl version --client
            kubectl config get-contexts || true
          '''
        }
      }
    }

    stage('Validate (kustomize)') {
      when { expression { fileExists(env.KUSTOMIZE_DIR) } }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh 'kubectl kustomize ${KUSTOMIZE_DIR} > /dev/null'
        }
      }
    }

    stage('Apply') {
      when { expression { fileExists(env.KUSTOMIZE_DIR) } }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            kubectl apply -k ${KUSTOMIZE_DIR}
            kubectl -n ${KUBE_NS} get pods -o wide
          '''
        }
      }
    }
  }

  post {
    success {
      withCredentials([
        file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
        string(credentialsId: 'telegram-bot-token', variable: 'TG')
      ]) {
        sh '''
          set -e
          NODEIP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[?(@.type=='InternalIP')].address}")
          NODEPORT=$(kubectl -n ${KUBE_NS} get svc ${REGISTRY_SVC} -o jsonpath="{.spec.ports[0].nodePort}" || echo 30500)
          TEXT="✅ Deploy OK: ${JOB_NAME} #${BUILD_NUMBER}\\nRegistry: http://${NODEIP}:${NODEPORT}/v2/_catalog"

          curl -s -X POST -H 'Content-Type: application/json' --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
          {"chat_id":"${TG_CHAT}","text":"${TEXT}"}
EOF
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        // ВАЖНО: '|| true' теперь внутри скрипта, либо можно use returnStatus:true
        sh(script: '''
          curl -s -X POST -H 'Content-Type: application/json' --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
          {"chat_id":"${TG_CHAT}","text":"❌ Deploy FAILED: ${JOB_NAME} #${BUILD_NUMBER} (см. логи Jenkins)"}
EOF
        ''', returnStatus: true)
      }
    }
  }
}
