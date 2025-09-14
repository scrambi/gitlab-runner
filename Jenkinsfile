pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  environment {
    NAMESPACE   = 'gitlab-runner'
    RELEASE     = 'gitlab-runner'
    GITLAB_URL  = 'http://192.168.31.125/'   // URL твоего GitLab (HTTP без TLS ок)
    TG_CHAT     = '180424264'                // твой chat_id
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Kubectl/Helm version') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            set -e
            kubectl version --client
            kubectl config get-contexts || true
            if ! command -v helm >/dev/null 2>&1; then
              echo "Installing helm…"
              curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
            fi
            helm version
          '''
        }
      }
    }

    stage('Install/Upgrade GitLab Runner (Helm)') {
      steps {
        withCredentials([
          file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
          string(credentialsId: 'gitlab-runner-token', variable: 'RUNNER_TOKEN')
        ]) {
          sh '''
            set -euo pipefail

            # ns
            kubectl get ns ${NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${NAMESPACE}

            # helm repo
            helm repo add gitlab https://charts.gitlab.io >/dev/null 2>&1 || true
            helm repo update

            # deploy/upgrade
            helm upgrade --install ${RELEASE} gitlab/gitlab-runner \
              --namespace ${NAMESPACE} \
              --set gitlabUrl="${GITLAB_URL}" \
              --set runnerRegistrationToken="${RUNNER_TOKEN}" \
              --set runners.tags="k8s" \
              --set runners.runUntagged=true \
              --set runners.privileged=true \
              --set rbac.create=true

            # показать состояние
            kubectl -n ${NAMESPACE} get pods -o wide
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
          TEXT="✅ GitLab Runner deployed: ${JOB_NAME} #${BUILD_NUMBER}\\nURL: ${GITLAB_URL}\\nTag: k8s"
          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
          {"chat_id":"${TG_CHAT}","text":"${TEXT}"}
EOF
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'TG')]) {
        sh(script: '''
          curl -s -X POST -H 'Content-Type: application/json' \
            --data-binary @- "https://api.telegram.org/bot${TG}/sendMessage" <<EOF
          {"chat_id":"''' + "${TG_CHAT}" + '''","text":"❌ Deploy FAILED: ${JOB_NAME} #${BUILD_NUMBER} (см. логи Jenkins)"}
EOF
        ''', returnStatus: true)
      }
    }
  }
}
