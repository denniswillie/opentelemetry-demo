pipeline {
  agent { kubernetes { yaml """apiVersion: v1
kind: Pod
metadata: { labels: { app: jenkins-agent } }
spec:
  containers:
  - name: docker
    image: docker:25
    tty: true
    securityContext: { privileged: true }
  """ } }

  environment {
    IMAGE_TAG = "local"
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
          script {
            env.CHANGED_SERVICES.split().each { svc ->
              dir("src/${svc}") {
                script {
                  def buildCtx = '.'
                  if (!fileExists('Dockerfile')) {
                    if (fileExists('src/Dockerfile')) {
                      buildCtx = 'src'
                    } else {
                      echo "Skipping ${svc}: no Dockerfile found in expected locations"
                      return
                    }
                  }

                  sh """
                    docker build -t ${svc}:${IMAGE_TAG} ${buildCtx}
                  """
                }
              }
            }
          }
        }
      }
    }

    stage('Deploy via Helm') {
      when {
        expression { return env.CHANGED_SERVICES }
      }
      steps {
        sh '''
            helm upgrade --install otel-demo $CHART \
            --namespace $KUBE_NS \
            --set image.repository=${svc} \
            --set image.tag=${IMAGE_TAG} \
            --set image.pullPolicy=Never
        '''
      }
    }
  }

  post { success { echo "✔ Deployed commit ${env.GIT_COMMIT} to Minikube" } }
}
