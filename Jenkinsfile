pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  serviceAccountName: jenkins-helm
  containers:
  - name: docker                 # ⬅ keeps the old image-build steps working
    image: docker:25
    tty: true
    securityContext:
      privileged: true
  - name: helm                   # ⬅ brand-new container with helm installed
    image: alpine/helm:3.14.4    # Any image that has helm 3 is fine
    command: ["cat"]             # Keeps the container alive waiting for Jenkins
    tty: true
"""
    }
  }

  environment {
    IMAGE_TAG = "latest"
    CHART = "open-telemetry/opentelemetry-demo"
    KUBE_NS = "otel-demo"
  }

  stages {
    stage('Checkout')  { steps { checkout scm } }

    stage('Determine Changed Services') {
      steps {
        script {
          // Identify files changed since the previous commit (fallback to HEAD~1 if none)
          def baseRef = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT ?: 'HEAD~1'
          def changedFiles = sh(returnStdout: true, script: "git diff --name-only ${baseRef} HEAD").trim().split("\n")

          def services = [] as Set
          changedFiles.each { path ->
            def m = path =~ /^src\\/([^\\/]+)\\//
            if (m) {
              services << m[0][1]
            }
          }
          env.CHANGED_SERVICES = services.join(' ')
          echo "Detected changed services: ${env.CHANGED_SERVICES ?: 'none'}"
        }
      }
    }

    stage('Build images for changed services') {
      when {
        expression { return env.CHANGED_SERVICES }
      }
      steps {
        container('docker') {
          sh """
            docker compose -f docker-compose.yml build ${env.CHANGED_SERVICES}
          """
        }
      }
    }

    stage('Deploy via Helm') {
      when {
        expression { return env.CHANGED_SERVICES }
      }
      steps {
        container('helm') {        // run the helm command in the new container
          sh """
            # Ensure repo present
            helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts || true
            helm repo update
            helm upgrade --install otel-demo $CHART \
              --namespace $KUBE_NS \
              --set default.image.tag=$IMAGE_TAG \
              --set default.image.pullPolicy=Never
          """
        }
      }
    }
  }

  post { success { echo "✔ Deployed commit ${env.GIT_COMMIT} to Minikube" } }
}
