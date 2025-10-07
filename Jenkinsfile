// Jenkinsfile (multibranch)
pipeline {
  agent { label 'docker' } // ensure agent has docker CLI & docker daemon or docker-in-docker
  options {
    buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '50'))
    disableConcurrentBuilds()
    timestamps()
  }

  environment {
    // Fill these in Jenkins credentials/global config
    DOCKERHUB_CREDENTIALS = 'dockerhub-creds'   // Jenkins username/password credential id
    DOCKERHUB_USER = '<dockerhub-username>'     // replace or set via Jenkins global env
    IMAGE_NAME = 'websiteburger'                // image name on Docker Hub
    GITHUB_TOKEN_CREDENTIALS = 'github-token'   // Jenkins secret text containing a GitHub token
    GITHUB_API_URL = 'https://api.github.com'
    K8S_DEPLOY_JOB = 'k8s-deploy'               // downstream Jenkins job name that performs helm deploy
    JIRA_CREDENTIALS = 'jira-creds'             // user:api_token (basic) or username/password credential id
    JIRA_BASE = 'https://yourcompany.atlassian.net' // change to your JIRA base URL
    HELM_RELEASE = 'websiteburger'              // default Helm release name
    HELM_NAMESPACE = 'default'                  // k8s namespace to deploy into
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set Version') {
      steps {
        script {
          // Compute simple version: major.minor.patch style using git commit count as patch.
          // You can adapt to use a VERSION file if you prefer.
          def commitCount = sh(script: "git rev-list --count HEAD", returnStdout: true).trim()
          def shortSha = sh(script: "git rev-parse --short=7 HEAD", returnStdout: true).trim()
          // Example version: 0.0.<commitCount>  -> user asked about 0.01 style; you can change major.minor
          env.IMAGE_TAG = "0.0.${commitCount}"
          env.IMAGE_SHA = shortSha
          echo "IMAGE_TAG=${env.IMAGE_TAG} IMAGE_SHA=${env.IMAGE_SHA}"
        }
      }
    }

    stage('Notify GitHub - PENDING') {
      steps {
        script {
          // for multibranch/PR builds, set status on the commit
          def token = credentials(env.GITHUB_TOKEN_CREDENTIALS)
          def payload = [
            state: "pending",
            description: "Jenkins: building",
            context: "jenkins/multibranch"
          ]
          sh """
            curl -s -X POST -H "Authorization: token ${token}" -H "Content-Type: application/json" \
              -d '${groovy.json.JsonOutput.toJson(payload)}' \
              ${env.GITHUB_API_URL}/repos/${env.GIT_URL.split(':')[-1].replace('.git','')}/statuses/$(git rev-parse HEAD)
          """
        }
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        script {
          docker.withRegistry("https://index.docker.io/v1/", env.DOCKERHUB_CREDENTIALS) {
            // Build image
            def imageFull = "${env.DOCKERHUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
            echo "Building ${imageFull}"
            def img = docker.build(imageFull, "--pull .")
            // push
            img.push()
            // also push 'latest' if you want (optional)
            echo "Pushed ${imageFull}"
            // set env for downstream
            env.PUSHED_IMAGE = imageFull
          }
        }
      }
    }

    stage('Trigger Deploy Job') {
      steps {
        script {
          // Trigger a downstream Pipeline job that runs Helm. Pass IMAGE and TAG.
          // The downstream job can be a Pipeline job that uses helm/kubectl with a kubeconfig credential.
          def params = [
            string(name: 'IMAGE', value: env.PUSHED_IMAGE),
            string(name: 'IMAGE_TAG', value: env.IMAGE_TAG),
            string(name: 'HELM_RELEASE', value: env.HELM_RELEASE),
            string(name: 'HELM_NAMESPACE', value: env.HELM_NAMESPACE)
          ]
          // Synchronously wait for the job so we can report result to JIRA/GitHub
          def buildInfo = build job: env.K8S_DEPLOY_JOB, parameters: params, wait: true, propagate: false
          echo "Deploy job result: ${buildInfo.getResult()}"
          env.DEPLOY_RESULT = buildInfo.getResult()
        }
      }
    }

    stage('Post-deploy updates (GitHub/JIRA)') {
      steps {
        script {
          // Update GitHub status
          def token = credentials(env.GITHUB_TOKEN_CREDENTIALS)
          def state = (env.DEPLOY_RESULT == 'SUCCESS') ? 'success' : 'failure'
          def desc = (env.DEPLOY_RESULT == 'SUCCESS') ? "Deployed to ${env.HELM_NAMESPACE}" : "Deploy failed"
          def payload = [ state: state, description: desc, context: "jenkins/deploy" ]
          sh """
            curl -s -X POST -H "Authorization: token ${token}" -H "Content-Type: application/json" \
              -d '${groovy.json.JsonOutput.toJson(payload)}' \
              ${env.GITHUB_API_URL}/repos/${env.GIT_URL.split(':')[-1].replace('.git','')}/statuses/$(git rev-parse HEAD)
          """

          // Update JIRA: If branch or PR title includes JIRA-KEY (e.g. PROJ-123), pick first occurrence.
          def jiraKey = ""
          // check branch name
          def branch = env.BRANCH_NAME ?: sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
          def m = (branch =~ /([A-Z]{2,}-\\d+)/)
          if (m.size() > 0) {
            jiraKey = m[0][1]
          } else {
            // try commit message or PR title (if available via env)
            def commitMsg = sh(script: "git log -1 --pretty=%B", returnStdout: true).trim()
            def mm = (commitMsg =~ /([A-Z]{2,}-\\d+)/)
            if (mm.size() > 0) { jiraKey = mm[0][1] }
          }

          if (jiraKey) {
            echo "Found JIRA key: ${jiraKey}, updating issue"
            withCredentials([usernamePassword(credentialsId: env.JIRA_CREDENTIALS, passwordVariable: 'JIRA_PASS', usernameVariable: 'JIRA_USER')]) {
              def comment = "${env.JOB_NAME}: Build ${currentBuild.number} - ${env.DEPLOY_RESULT}. Image: ${env.PUSHED_IMAGE}"
              sh """
                curl -s -u ${JIRA_USER}:${JIRA_PASS} -X POST -H 'Content-Type: application/json' \
                  --data '{ "body": "${comment}" }' \
                  ${env.JIRA_BASE}/rest/api/2/issue/${jiraKey}/comment
              """
              // Optionally transition issue if deploy succeeded (transition id depends on your workflow)
              if (env.DEPLOY_RESULT == 'SUCCESS') {
                // NOTE: replace transition id below with your JIRA's transition ID for "Deployed" status
                def transitionId = '31'
                sh """
                  curl -s -u ${JIRA_USER}:${JIRA_PASS} -X POST -H 'Content-Type: application/json' \
                    --data '{ "transition": { "id": "${transitionId}" } }' \
                    ${env.JIRA_BASE}/rest/api/2/issue/${jiraKey}/transitions
                """
              }
            }
          } else {
            echo "No JIRA key found in branch/commit message; skipping JIRA update."
          }
        } // script
      } // steps
    } // stage
  } // stages

  post {
    failure {
      script {
        // If overall pipeline fails, post a failing GitHub status
        def token = credentials(env.GITHUB_TOKEN_CREDENTIALS)
        def payload = [ state: "failure", description: "Jenkins pipeline failed", context: "jenkins/multibranch" ]
        sh """
          curl -s -X POST -H "Authorization: token ${token}" -H "Content-Type: application/json" \
            -d '${groovy.json.JsonOutput.toJson(payload)}' \
            ${env.GITHUB_API_URL}/repos/${env.GIT_URL.split(':')[-1].replace('.git','')}/statuses/$(git rev-parse HEAD)
        """
      }
    }
  }
}
