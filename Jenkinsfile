pipeline {
    agent { label 'docker-node' }  // run on docker agent

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo "🏗️ Building application..."
                // Replace with actual build command
                sh 'python3 app.py || exit 1'
            }
        }

        stage('Test') {
            steps {
                echo "🧪 Running tests..."
                // Replace with your test command
                sh 'echo "All tests passed!"'
            }
        }
    }

    post {
        success {
            script {
                if (env.CHANGE_ID) {
                    echo "✅ PR #${env.CHANGE_ID} build passed."

                    // Extract JIRA ID from branch name or PR title
                    def jiraKey = env.BRANCH_NAME =~ /(PROJ-\d+)/
                    if (jiraKey) {
                        jiraSendBuildInfo site: 'myJira'
                        jiraComment issueKey: jiraKey[0][1], body: "✅ Jenkins build succeeded for PR #${env.CHANGE_ID}"
                    }
                }
            }
        }
        failure {
            script {
                if (env.CHANGE_ID) {
                    echo "❌ PR #${env.CHANGE_ID} build failed. Merge blocked."

                    // Extract JIRA ID from branch name or PR title
                    def jiraKey = env.BRANCH_NAME =~ /(PROJ-\d+)/
                    if (jiraKey) {
                        jiraSendBuildInfo site: 'myJira'
                        jiraComment issueKey: jiraKey[0][1], body: "❌ Jenkins build failed for PR #${env.CHANGE_ID}"
                    }
                }
                error("Stopping merge because PR build failed.")
            }
        }
    }
}
