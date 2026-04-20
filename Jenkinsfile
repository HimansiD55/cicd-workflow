pipeline {
    agent { label 'ec2-reviewer' }

    options {
        timeout(time: 20, unit: 'MINUTES')
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
                    def segs = (env.GIT_URL ?: scm.userRemoteConfigs[0].url ?: '').tokenize('/')
                    if (segs.size() < 2) error("Cannot parse repo from URL")
                    env.REPO_PATH   = "${segs[-2]}/${segs[-1].replace('.git', '')}"
                    env.PR_NUMBER   = env.CHANGE_ID
                    env.BASE_BRANCH = env.CHANGE_TARGET ?: 'main'
                    env.PR_HEAD_SHA = ''
                }
            }
        }

        stage('Checkout Full Repo') {
            steps {
                script {
                    checkout([$class: 'GitSCM',
                        branches: [[name: "origin/pr/${env.PR_NUMBER}/head"]],
                        extensions: [],
                        userRemoteConfigs: [[
                            url: env.GIT_URL ?: scm.userRemoteConfigs[0].url,
                            refspec: "+refs/pull/${env.PR_NUMBER}/head:refs/remotes/origin/pr/${env.PR_NUMBER}/head",
                            credentialsId: 'github-token'
                        ]]
                    ])
                    env.PR_HEAD_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    sh "git fetch origin ${env.BASE_BRANCH}:refs/remotes/origin/${env.BASE_BRANCH}"
                }
            }
        }

        stage('Build Context Package') {
            steps {
                script {
                    def diff = sh(script: "git diff origin/${env.BASE_BRANCH}...HEAD -- ':!*.lock' ':!package-lock.json' ':!yarn.lock'", returnStdout: true).trim()
                    writeFile file: 'pr_diff.txt', text: diff ?: "No changes"

                    def changedFiles = sh(script: "git diff --name-only origin/${env.BASE_BRANCH}...HEAD -- ':!*.lock' ':!package-lock.json' ':!yarn.lock'", returnStdout: true).trim().split('\n').findAll { it }
                    env.CHANGED_FILES_LIST = changedFiles.join(', ')

                    def context = new StringBuilder("=== FULL CONTENTS OF CHANGED FILES ===\n\n")
                    changedFiles.each { f ->
                        if (fileExists(f)) {
                            def c = readFile(f)
                            context.append("--- FILE: ${f} ---\n${c.size() > 30000 ? c.take(30000) + '\n...[truncated]' : c}\n\n")
                        }
                    }
                    writeFile file: 'full_context.txt', text: context.toString()
                }
            }
        }

        stage('Claude Review') {
            steps {
                script {
                    def prompt = """You are a senior code reviewer with full access to the repository.
CHANGED FILES: ${env.CHANGED_FILES_LIST}

=== GIT DIFF ===
${readFile('pr_diff.txt')}

${readFile('full_context.txt')}

Return ONLY a JSON object, no markdown fences:
{
  "verdict": "PASS or FAIL",
  "summary": "2-3 sentence summary",
  "issues": [{"file":"...","line":1,"severity":"critical|high|medium|low","title":"...","detail":"..."}]
}
If no issues, return empty issues array with verdict PASS."""

                    // Write prompt to file — avoid all shell string interpolation issues
                    writeFile file: 'prompt.txt', text: prompt

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=$CL_KEY
                            # Use --file flag to pass prompt safely — no shell expansion of contents
                            claude -p "$(cat prompt.txt)" --output-format json > claude_raw.json 2>&1 || true
                            # Extract and clean the result
                            jq -r '.result // empty' claude_raw.json | sed 's/^```json//; s/^```//; s/```$//' > review.json
                            # Validate it's real JSON
                            jq empty review.json || (echo "Claude returned invalid JSON:" && cat claude_raw.json && exit 1)
                        '''
                    }

                    env.REVIEW_VERDICT = sh(script: 'jq -r \'.verdict // "UNKNOWN"\' review.json', returnStdout: true).trim()
                }
            }
        }

        stage('Post to GitHub') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                    sh '''
                        VERDICT=$(jq -r '.verdict' review.json)
                        SUMMARY=$(jq -r '.summary' review.json)
                        ISSUE_COUNT=$(jq '.issues | length' review.json)
                        ICON="✅"; [ "$VERDICT" = "FAIL" ] && ICON="❌"

                        # Post main PR comment
                        ISSUES_TABLE=$(jq -r '.issues[] | "| " + .severity + " | `" + .file + "` L" + (.line|tostring) + " | **" + .title + "** — " + .detail + " |"' review.json 2>/dev/null || echo "")

                        jq -n --arg b "$ICON ## Claude AI Review: $VERDICT\n\n**Summary:** $SUMMARY\n\n**Issues found:** $ISSUE_COUNT\n\n| Severity | Location | Detail |\n|---|---|---|\n$ISSUES_TABLE" \
                            '{body: $b}' > pr_comment.json

                        curl -sf -X POST \
                             -H "Authorization: token $TKN" \
                             -H "Content-Type: application/json" \
                             "https://api.github.com/repos/${REPO_PATH}/issues/${PR_NUMBER}/comments" \
                             -d @pr_comment.json

                        # Post inline comments per issue
                        i=0
                        while [ $i -lt $ISSUE_COUNT ]; do
                            jq -c ".issues[$i]" review.json > /tmp/issue.json
                            LINE=$(jq -r '.line' /tmp/issue.json)
                            # Skip if line is not a valid integer
                            echo "$LINE" | grep -qE '^[0-9]+$' || { i=$((i+1)); continue; }

                            jq -n \
                                --arg body "[$(jq -r '.severity' /tmp/issue.json)] $(jq -r '.title' /tmp/issue.json)\n\n$(jq -r '.detail' /tmp/issue.json)" \
                                --arg path "$(jq -r '.file' /tmp/issue.json)" \
                                --arg sha "${PR_HEAD_SHA}" \
                                --argjson line "$LINE" \
                                '{commit_id:$sha, path:$path, line:$line, side:"RIGHT", body:$body}' \
                            | curl -sf -X POST \
                                   -H "Authorization: token $TKN" \
                                   -H "Content-Type: application/json" \
                                   "https://api.github.com/repos/${REPO_PATH}/pulls/${PR_NUMBER}/comments" \
                                   -d @- || true

                            i=$((i+1))
                        done
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                if (env.PR_HEAD_SHA) {
                    def state = (env.REVIEW_VERDICT == 'PASS') ? 'success' : 'failure'
                    withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                        sh """
                            curl -sf -X POST \
                                 -H "Authorization: token \$TKN" \
                                 -H "Content-Type: application/json" \
                                 -d '{"state":"${state}","context":"claude-ai-review","description":"Verdict: ${env.REVIEW_VERDICT}"}' \
                                 "https://api.github.com/repos/${env.REPO_PATH}/statuses/${env.PR_HEAD_SHA}" || true
                        """
                    }
                }
            }
            cleanWs()
        }
    }
}
