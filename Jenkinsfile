pipeline {
    agent { label 'ec2-reviewer' }

    options {
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        GITHUB_TOKEN   = credentials('github-token')
        CLAUDE_API_KEY = credentials('ANTHROPIC_API_KEY')
    }

    stages {
        stage('Validate PR Context') {
            steps {
                script {
                    if (!env.CHANGE_ID) {
                        currentBuild.result = 'NOT_BUILT'
                        error("Skipping non-PR build")
                    }

                    def gitUrl = env.GIT_URL ?: scm.userRemoteConfigs[0].url ?: ''
                    def repoSegments = gitUrl.tokenize('/')
                    env.REPO_OWNER = repoSegments[-2]
                    env.REPO_NAME  = repoSegments[-1].replace('.git', '')
                    env.REPO_PATH  = "${env.REPO_OWNER}/${env.REPO_NAME}"
                    env.PR_NUMBER  = env.CHANGE_ID
                    env.BASE_BRANCH = env.CHANGE_TARGET ?: 'main'
                }
            }
        }

        stage('Checkout & Diff') {
            steps {
                script {
                    checkout([$class: 'GitSCM', 
                        branches: [[name: "origin/pr/${env.PR_NUMBER}/head"]],
                        userRemoteConfigs: [[url: env.GIT_URL ?: scm.userRemoteConfigs[0].url, 
                        refspec: "+refs/pull/${env.PR_NUMBER}/head:refs/remotes/origin/pr/${env.PR_NUMBER}/head", 
                        credentialsId: 'github-token']]
                    ])
                    env.PR_HEAD_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    
                    sh "git fetch origin ${env.BASE_BRANCH}:refs/remotes/origin/${env.BASE_BRANCH} --depth=50"
                    def diff = sh(script: "git diff origin/${env.BASE_BRANCH}...HEAD -- ':!*.lock' ':!package-lock.json' ':!yarn.lock' | head -c 80000", returnStdout: true).trim()
                    
                    writeFile file: 'pr_diff.txt', text: diff ?: "No changes"
                }
            }
        }

        stage('Claude Review') {
            steps {
                script {
                    def diffText = readFile('pr_diff.txt')
                    writeFile file: 'prompt.txt', text: "Review this code. Return ONLY JSON. Schema: { \"summary\": \"\", \"verdict\": \"PASS/FAIL\", \"critical_issues\": [], \"warnings\": [] }. \n\n ${diffText}"

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=$CL_KEY
                            # Run Claude, peel the envelope, and save to review_result.json
                            claude -p "$(cat prompt.txt)" --output-format json | jq -r '.result' > review_result.json
                        '''
                    }
                    
                    // Read the file we just created
                    def rawJson = readFile('review_result.json').trim()
                    
                    // Clean up any extra markdown backticks if Claude added them
                    def cleanJson = rawJson.replaceAll("```json", "").replaceAll("```", "").trim()
                    
                    // Overwrite the file with the clean version so the next stage finds it
                    writeFile file: 'review_result.json', text: cleanJson
                    
                    // Get the verdict for the GitHub Status
                    env.REVIEW_VERDICT = sh(script: "jq -r '.verdict // \"UNKNOWN\"' review_result.json", returnStdout: true).trim()
                }
            }
        }

        stage('Post to GitHub') {
            steps {
                script {
                    // This now looks for 'review_result.json', which we know exists
                    def jsonText = readFile('review_result.json').trim()
                    def data = new groovy.json.JsonSlurper().parseText(jsonText)

                    def emoji = (data.verdict == 'PASS') ? "✅" : "❌"
                    def markdown = "## ${emoji} Claude AI Code Review: ${data.verdict}\n\n### 📝 Summary\n${data.summary}\n\n"
                    
                    if (data.critical_issues) {
                        markdown += "### ⚠️ Critical Issues\n"
                        data.critical_issues.each { issue ->
                            markdown += "- **${issue.category}**: ${issue.issue} (File: ${issue.file})\n"
                        }
                    }

                    def payload = groovy.json.JsonOutput.toJson([body: markdown])
                    writeFile file: 'github_payload.json', text: payload

                    withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                        sh '''
                            curl -sf -X POST \
                                 -H "Authorization: token $TKN" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 -H "Content-Type: application/json" \
                                 "https://api.github.com/repos/${REPO_PATH}/issues/${PR_NUMBER}/comments" \
                                 -d @github_payload.json
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Ensure variables exist before calling curl
                if (env.REPO_PATH && env.PR_HEAD_SHA) {
                    def state = (env.REVIEW_VERDICT == 'PASS') ? 'success' : 'failure'
                    withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                        sh """
                            curl -sf -X POST -H "Authorization: token \$TKN" \
                            -d '{"state":"${state}","context":"claude-ai-review"}' \
                            "https://api.github.com/repos/${env.REPO_PATH}/statuses/${env.PR_HEAD_SHA}" || true
                        """
                    }
                }
            }
            cleanWs()
        }
    }
}
