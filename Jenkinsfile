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
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
  serviceAccountName: jenkins-helm
  containers:
  - name: docker
    image: docker:25-cli
    tty: true
    env:
    - name: DOCKER_HOST         # tell the CLI where the daemon is
      value: unix:///var/run/docker.sock
    - name: DOCKER_TLS_CERTDIR  # disable the dind entry-point logic if present
      value: ""
    securityContext:
      privileged: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  - name: helm
    image: alpine/helm:3   # Any image that has helm 3 is fine
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
    stage('Checkout')  {
      steps {
        container('jnlp') {
          checkout scm
        }
      }
    }

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
        container('helm') {
          script {
            def servicesArgs = env.CHANGED_SERVICES.split().collect { svc ->
              "--set components.${svc}.imageOverride.tag=${IMAGE_TAG}-${svc}"
            }.join(' ')

            sh """
              # Ensure repo present
              helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts || true
              helm repo update
              helm upgrade --install otel-demo $CHART \
                --namespace $KUBE_NS \
                --reuse-values \
                ${servicesArgs}
            """
          }
        }
      }
    }
  }

  post { success { echo "✔ Deployed commit ${env.GIT_COMMIT} to Minikube" } }
}
