pipeline {
    agent { label 'ec2-reviewer' }

    options {
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        // These are used for global access
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
                    
                    // FIXED: Added safety check to prevent crash
                    if (repoSegments.size() >= 2) {
                        env.REPO_OWNER = repoSegments[-2]
                        env.REPO_NAME  = repoSegments[-1].replace('.git', '')
                        env.REPO_PATH  = "${env.REPO_OWNER}/${env.REPO_NAME}"
                    } else {
                        error("Could not parse repository owner and name from URL: ${gitUrl}")
                    }
                    
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
                    writeFile file: 'prompt.txt', text: "Review this code. Return ONLY JSON with fields: summary, verdict (PASS/FAIL). \n\n ${diffText}"

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=$CL_KEY
                            # Use jq to peel the envelope and handle markdown fences automatically
                            claude -p "$(cat prompt.txt)" --output-format json | jq -r '.result' | sed 's/```json//g' | sed 's/```//g' > review.json
                        '''
                    }
                    // Extract verdict for the post-build status
                    env.REVIEW_VERDICT = sh(script: "jq -r '.verdict // \"UNKNOWN\"' review.json", returnStdout: true).trim()
                }
            }
        }

        stage('Post to GitHub') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                    sh '''
                        # Build the JSON payload using JQ (Safe and Easy)
                        jq -n --slurpfile r review.json \
                        '{body: ("## Claude AI Review: " + $r[0].verdict + "\n\n### Summary\n" + $r[0].summary)}' > payload.json

                        curl -sf -X POST \
                             -H "Authorization: token $TKN" \
                             -H "Content-Type: application/json" \
                             "https://api.github.com/repos/${REPO_PATH}/issues/${PR_NUMBER}/comments" \
                             -d @payload.json
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                if (env.REPO_PATH && env.PR_HEAD_SHA) {
                    // This creates the Red/Green status in your PR console
                    def state = (env.REVIEW_VERDICT == 'PASS') ? 'success' : 'failure'
                    withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                        sh """
                            curl -sf -X POST -H "Authorization: token \$TKN" \
                            -d '{"state":"${state}","context":"claude-ai-review","description":"Review Verdict: ${env.REVIEW_VERDICT}"}' \
                            "https://api.github.com/repos/${env.REPO_PATH}/statuses/${env.PR_HEAD_SHA}" || true
                        """
                    }
                }
            }
            cleanWs()
        }
    }
}
