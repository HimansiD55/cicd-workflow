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
                    // Explicitly tell Claude NOT to use markdown fences
                    writeFile file: 'prompt.txt', text: "Perform a code review. Return ONLY a raw JSON object. Do NOT use markdown code blocks or backticks. Schema: { \"summary\": \"\", \"verdict\": \"PASS/FAIL\", \"critical_issues\": [], \"warnings\": [] }. \n\n ${diffText}"

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=$CL_KEY
                            
                            # 1. Get the result
                            # 2. Extract the .result field
                            # 3. Strip any accidental markdown fences (```json or ```)
                            claude -p "$(cat prompt.txt)" --output-format json | \
                            jq -r '.result' | \
                            sed 's/^```json//g' | sed 's/^```//g' | sed 's/```$//g' > clean_review.json
                        '''
                    }
                    
                    // Verify the file isn't empty before running jq again
                    def cleanReview = readFile('clean_review.json').trim()
                    if (cleanReview) {
                        env.REVIEW_VERDICT = sh(script: "jq -r '.verdict // \"UNKNOWN\"' clean_review.json", returnStdout: true).trim()
                    } else {
                        env.REVIEW_VERDICT = "UNKNOWN"
                    }
                    
                    echo "Verdict captured: ${env.REVIEW_VERDICT}"
                }
            }
        }

        stage('Post to GitHub') {
            steps {
                script {
                    // 1. Read and parse the JSON file
                    def jsonText = readFile('final_review.json').trim()
                    def data = new groovy.json.JsonSlurper().parseText(jsonText)

                    // 2. Build a Clean Markdown Report
                    def emoji = (data.verdict == 'PASS') ? "✅" : "❌"
                    def markdown = """
## ${emoji} Claude AI Code Review: ${data.verdict}

### 📝 Summary
${data.summary}

### ⚠️ Issues Found
"""
                    // Add critical issues to the report
                    data.critical_issues.each { issue ->
                        markdown += "- **[CRITICAL]** ${issue.file}: ${issue.issue}\n"
                        markdown += "  *Fix:* ${issue.recommendation}\n\n"
                    }

                    markdown += "\n--- \n*Sent from Jenkins via Claude AI*"

                    // 3. Post the Clean Markdown
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
