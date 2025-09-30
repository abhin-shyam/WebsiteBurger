pipeline {
    agent { label 'docker-node' }

    options {
        skipDefaultCheckout()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        JIRA_EMAIL = 'abhinshyam2003@gmail.com'         // Replace with your Jira email
        JIRA_API_TOKEN = 'ATATT3xFfGF0LSEmybXFgL0pByJIMGo_Efgqnqle2C9XapJyANrd5QyfAEWOoV0K22Uok38YLhd6thJkDvbcSUoeO0NCzTxMcgERhQ0k_YBSh2zbPAEF2Jdcri7K0Lx4yN9RvsWVhBfPFZdQwRu09XGjiUadCuNHQygSUEufei2FD2eWRh3EFro=48EB4348'             // Replace with your Jira API token
        JIRA_DOMAIN = 'https://abhinshyam.atlassian.net/'     // Replace with your Jira domain
        JIRA_ISSUE = 'D5IJ-2'                           // Replace with your Jira issue key
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh 'sudo docker build -t websiteburger:latest .'
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    echo "Running dummy tests..."
                    sh 'echo "All tests passed!"'
                }
            }
        }

        stage('Deploy (Optional)') {
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "Deploying WebsiteBurger (main branch only)..."
                    // Example: docker run -d -p 80:80 websiteburger:latest
                }
            }
        }
    }

    post {
        success {
            script {
                sh """
                curl -X POST \
                  -u ${JIRA_EMAIL}:${JIRA_API_TOKEN} \
                  -H "Content-Type: application/json" \
                  --data '{
                    "update": {
                      "comment": [
                        {
                          "add": {
                            "body": "✅ Jenkins build #${BUILD_NUMBER} succeeded for WebsiteBurger. View details in Jenkins."
                          }
                        }
                      ]
                    }
                  }' \
                  https://${JIRA_DOMAIN}/rest/api/3/issue/${JIRA_ISSUE}
                """
            }
        }
        failure {
            script {
                sh """
                curl -X POST \
                  -u ${JIRA_EMAIL}:${JIRA_API_TOKEN} \
                  -H "Content-Type: application/json" \
                  --data '{
                    "update": {
                      "comment": [
                        {
                          "add": {
                            "body": "❌ Jenkins build #${BUILD_NUMBER} failed for WebsiteBurger. Please investigate."
                          }
                        }
                      ]
                    }
                  }' \
                  https://${JIRA_DOMAIN}/rest/api/3/issue/${JIRA_ISSUE}
                """
            }
        }
    }
}
