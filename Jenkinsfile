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
                    writeFile file: 'prompt.txt', text: "Review this code and return ONLY JSON:\n${diffText}"

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=$CL_KEY
                            claude -p "$(cat prompt.txt)" --output-format json > review.json
                        '''
                    }
                    def review = readFile('review.json').trim()
                    env.REVIEW_VERDICT = sh(script: "jq -r '.verdict // \"UNKNOWN\"' review.json", returnStdout: true).trim()
                }
            }
        }

        stage('Post to GitHub') {
            steps {
                script {
                    // PREPARE DATA OUTSIDE OF SH
                    def reviewContent = readFile('review.json').trim()
                    def jsonPayload = groovy.json.JsonOutput.toJson([body: reviewContent])
                    writeFile file: 'payload.json', text: jsonPayload

                    withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                        sh '''
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
