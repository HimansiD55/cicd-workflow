
pipeline {
    agent {
        label 'ec2-reviewer'
    }

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
                        echo "Not a PR build (branch: ${env.BRANCH_NAME}). Skipping."
                        currentBuild.result = 'NOT_BUILT'
                        error("Skipping non-PR build — CHANGE_ID is not set")
                    }

                    def gitUrl = env.GIT_URL ?: scm.userRemoteConfigs[0].url ?: ''
                    def repoSegments = gitUrl.tokenize('/')
                    env.REPO_OWNER  = repoSegments.size() >= 2 ? repoSegments[-2] : ''
                    env.REPO_NAME   = repoSegments.size() >= 1 ? repoSegments[-1].replace('.git', '') : ''
                    env.REPO_PATH   = "${env.REPO_OWNER}/${env.REPO_NAME}"
                    env.PR_NUMBER   = env.CHANGE_ID ?: ''
                    env.BASE_BRANCH = env.CHANGE_TARGET ?: 'main'
                    env.HEAD_BRANCH = env.CHANGE_BRANCH ?: ''
                    env.PR_HEAD_SHA = env.GIT_COMMIT ?: ''

                    echo "Repo: ${env.REPO_PATH} | PR #${env.PR_NUMBER} | ${env.HEAD_BRANCH} -> ${env.BASE_BRANCH}"
                }
            }
        }

        stage('Checkout') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "origin/pr/${env.PR_NUMBER}/head"]],
                        userRemoteConfigs: [[
                            url: env.GIT_URL ?: scm.userRemoteConfigs[0].url,
                            refspec: "+refs/pull/${env.PR_NUMBER}/head:refs/remotes/origin/pr/${env.PR_NUMBER}/head",
                            credentialsId: 'github-token'
                        ]],
                        extensions: [
                            [$class: 'CloneOption', depth: 0, noTags: true, shallow: false]
                        ]
                    ])

                    env.PR_HEAD_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    echo "Reviewing commit: ${env.PR_HEAD_SHA}"
                }
            }
        }

        stage('Get PR Diff') {
            steps {
                script {
                    sh "git fetch origin ${env.BASE_BRANCH}:refs/remotes/origin/${env.BASE_BRANCH} --depth=50"

                    def diff = sh(
                        script: """
                            git diff origin/${env.BASE_BRANCH}...HEAD \
                                -- ':!*.lock' ':!package-lock.json' ':!yarn.lock' \
                                   ':!*.min.js' ':!*.min.css' ':!dist/' ':!build/' \
                                2>/dev/null | head -c 80000
                        """,
                        returnStdout: true
                    ).trim()

                    if (!diff) {
                        diff = "No meaningful code changes detected."
                    }

                    def changedFiles = sh(
                        script: "git diff --name-status origin/${env.BASE_BRANCH}...HEAD 2>/dev/null | head -100",
                        returnStdout: true
                    ).trim()

                    def prMetadata = sh(
                        script: """
                            curl -sf \
                                 -H "Authorization: token ${GITHUB_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 "https://api.github.com/repos/${env.REPO_PATH}/pulls/${env.PR_NUMBER}" \
                            | jq -r '{title: .title, body: (.body // "(no description)"), base: .base.ref, head: .head.ref}'
                        """,
                        returnStdout: true
                    ).trim()

                    writeFile file: '/tmp/pr_diff.txt',  text: diff
                    writeFile file: '/tmp/pr_files.txt', text: changedFiles
                    writeFile file: '/tmp/pr_meta.json', text: prMetadata

                    echo "Diff size: ${diff.size()} bytes"
                }
            }
        }

        stage('Setup Claude CLI') {
            steps {
                script {
                    def isInstalled = sh(
                        script: 'command -v claude >/dev/null 2>&1',
                        returnStatus: true
                    ) == 0

                    if (isInstalled) {
                        echo "Claude CLI already installed ✅"
                        sh 'claude --version || true'
                    } else {
                        echo "Claude CLI not found. Installing..."

                        sh '''
                            set -e

                            curl -fsSL https://claude.ai/install.sh -o /tmp/claude_install.sh

                            if [ ! -s /tmp/claude_install.sh ]; then
                                echo "ERROR: Failed to download installer"
                                exit 1
                            fi

                            chmod +x /tmp/claude_install.sh
                            bash /tmp/claude_install.sh

                            if ! command -v claude >/dev/null 2>&1; then
                                echo "ERROR: Claude installation failed"
                                exit 1
                            fi

                            echo "Claude installed successfully 🎉"
                            claude --version || true
                        '''
                    }
                }
            }
        }

        stage('Claude Native Review') {
            steps {
                script {
                    def diff         = readFile('/tmp/pr_diff.txt').trim()
                    def changedFiles = readFile('/tmp/pr_files.txt').trim()
                    def prMeta       = readFile('/tmp/pr_meta.json').trim()

                    writeFile file: '/tmp/prompt.txt', text: """You are a senior software engineer performing a code review.

## PR Metadata
${prMeta}

## Changed Files
${changedFiles}

## Code Diff
<diff>
${diff}
</diff>

Return ONLY valid JSON:

{
  "summary": "",
  "verdict": "PASS",
  "verdict_reason": "",
  "critical_issues": [],
  "warnings": [],
  "suggestions": [],
  "positive_notes": []
}
"""

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CLAUDE_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=${CLAUDE_KEY}

                            claude \
                                -p "$(cat /tmp/prompt.txt)" \
                                --permission-mode acceptEdits \
                                --allowedTools "Bash Read Edit Write" \
                                --output-format json \
                                --max-tokens 4096 \
                                > /tmp/review.json
                        '''
                    }

                    def reviewContent = readFile('/tmp/review.json').trim()
                    if (!reviewContent) {
                        error("Claude returned no content")
                    }

                    sh "jq -r '.verdict // \"UNKNOWN\"' /tmp/review.json > /tmp/verdict.txt || echo 'UNKNOWN' > /tmp/verdict.txt"
                    env.REVIEW_VERDICT = readFile('/tmp/verdict.txt').trim()

                    echo "Verdict: ${env.REVIEW_VERDICT}"
                }
            }
        }

        stage('Post PR Comment') {
            steps {
                script {
                    def commentBody = readFile('/tmp/review.json')

                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        sh """
                            curl -sf -X POST \
                                 -H "Authorization: token ${GH_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 "https://api.github.com/repos/${env.REPO_PATH}/issues/${env.PR_NUMBER}/comments" \
                                 -d '{"body": ${groovy.json.JsonOutput.toJson(commentBody)}}'
                        """
                    }
                }
            }
        }

        stage('Set PR Status') {
            steps {
                script {
                    def verdict = env.REVIEW_VERDICT ?: 'UNKNOWN'
                    def stateMap = [PASS: 'success', WARN: 'success', FAIL: 'failure', UNKNOWN: 'error']
                    def state = stateMap[verdict] ?: 'error'

                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        sh """
                            curl -sf -X POST \
                                 -H "Authorization: token ${GH_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 "https://api.github.com/repos/${env.REPO_PATH}/statuses/${env.PR_HEAD_SHA}" \
                                 -d '{
                                   "state": "${state}",
                                   "target_url": "${env.BUILD_URL}",
                                   "description": "Claude AI review: ${verdict}",
                                   "context": "claude-ai-review"
                                 }'
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f /tmp/*.txt /tmp/*.json || true'
            cleanWs()
        }

        success {
            script {
                if ((env.REVIEW_VERDICT ?: 'UNKNOWN') == 'FAIL') {
                    currentBuild.result = 'UNSTABLE'
                }
            }
        }

        failure {
            script {
                def repoPath = env.REPO_PATH ?: ''
                def sha      = env.PR_HEAD_SHA ?: ''

                if (repoPath && sha) {
                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        sh """
                            curl -sf -X POST \
                                 -H "Authorization: token ${GH_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 "https://api.github.com/repos/${repoPath}/statuses/${sha}" \
                                 -d '{"state":"error","description":"Jenkins pipeline error","context":"claude-ai-review"}' || true
                        """
                    }
                }
            }
        }
    }
}
