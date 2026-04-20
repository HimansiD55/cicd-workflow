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

                    def gitUrl = env.GIT_URL ?: scm.userRemoteConfigs[0].url ?: ''
                    def repoSegments = gitUrl.tokenize('/')

                    if (repoSegments.size() >= 2) {
                        env.REPO_OWNER = repoSegments[-2]
                        env.REPO_NAME  = repoSegments[-1].replace('.git', '')
                        env.REPO_PATH  = "${env.REPO_OWNER}/${env.REPO_NAME}"
                    } else {
                        error("Could not parse repo from URL: ${gitUrl}")
                    }

                    env.PR_NUMBER   = env.CHANGE_ID
                    env.BASE_BRANCH = env.CHANGE_TARGET ?: 'main'
                }
            }
        }

        stage('Checkout Full Repo') {
            steps {
                script {
                    // Full clone — no --depth, so Claude sees the whole repo
                    checkout([$class: 'GitSCM',
                        branches: [[name: "origin/pr/${env.PR_NUMBER}/head"]],
                        extensions: [], // no shallow clone extension
                        userRemoteConfigs: [[
                            url: env.GIT_URL ?: scm.userRemoteConfigs[0].url,
                            refspec: "+refs/pull/${env.PR_NUMBER}/head:refs/remotes/origin/pr/${env.PR_NUMBER}/head",
                            credentialsId: 'github-token'
                        ]]
                    ])

                    env.PR_HEAD_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()

                    // Fetch base branch fully too
                    sh "git fetch origin ${env.BASE_BRANCH}:refs/remotes/origin/${env.BASE_BRANCH}"
                }
            }
        }

        stage('Build Context Package') {
            steps {
                script {
                    // 1. Get the diff (still needed for focus)
                    def diff = sh(
                        script: "git diff origin/${env.BASE_BRANCH}...HEAD -- ':!*.lock' ':!package-lock.json' ':!yarn.lock'",
                        returnStdout: true
                    ).trim()
                    writeFile file: 'pr_diff.txt', text: diff ?: "No changes"

                    // 2. Get list of changed files
                    def changedFiles = sh(
                        script: "git diff --name-only origin/${env.BASE_BRANCH}...HEAD -- ':!*.lock' ':!package-lock.json' ':!yarn.lock'",
                        returnStdout: true
                    ).trim().split('\n').findAll { it }

                    // 3. For each changed file, collect its FULL current content
                    def fullFileContents = new StringBuilder()
                    fullFileContents.append("=== FULL CONTENTS OF CHANGED FILES ===\n\n")

                    changedFiles.each { filePath ->
                        if (fileExists(filePath)) {
                            def content = readFile(filePath)
                            // Cap each file at 30k chars to avoid token overflow
                            if (content.length() > 30000) {
                                content = content.take(30000) + "\n... [truncated] ..."
                            }
                            fullFileContents.append("--- FILE: ${filePath} ---\n")
                            fullFileContents.append(content)
                            fullFileContents.append("\n\n")
                        }
                    }

                    // 4. Find related files (files that import/reference the changed files)
                    def relatedContents = new StringBuilder()
                    relatedContents.append("=== RELATED FILES (import/reference changed files) ===\n\n")

                    changedFiles.each { changedFile ->
                        def baseName = changedFile.tokenize('/')[-1].replace('.js','').replace('.ts','').replace('.py','')
                        // Search repo for files that import/reference this file
                        def relatedFiles = sh(
                            script: """grep -rl "${baseName}" --include="*.js" --include="*.ts" --include="*.py" --include="*.java" . 2>/dev/null | grep -v node_modules | grep -v ".git" | head -5 || true""",
                            returnStdout: true
                        ).trim().split('\n').findAll { it && !changedFiles.contains(it.replace('./','')) }

                        relatedFiles.each { relPath ->
                            if (fileExists(relPath)) {
                                def content = readFile(relPath)
                                if (content.length() > 15000) content = content.take(15000) + "\n... [truncated] ..."
                                relatedContents.append("--- RELATED FILE: ${relPath} (references ${baseName}) ---\n")
                                relatedContents.append(content)
                                relatedContents.append("\n\n")
                            }
                        }
                    }

                    // 5. Write the full context package
                    writeFile file: 'full_context.txt', text: fullFileContents.toString() + relatedContents.toString()

                    // Save changed files list for prompt
                    env.CHANGED_FILES_LIST = changedFiles.join(', ')
                }
            }
        }

        stage('Claude Review') {
            steps {
                script {
                    def diffText      = readFile('pr_diff.txt')
                    def fullContext   = readFile('full_context.txt')

                    // Cap total prompt size (~100k chars safe limit)
                    def totalContext = fullContext.length() > 80000 ? fullContext.take(80000) + "\n...[context truncated]..." : fullContext

                    def prompt = """You are a senior code reviewer with full access to the repository.

CHANGED FILES IN THIS PR: ${env.CHANGED_FILES_LIST}

=== GIT DIFF (what changed) ===
${diffText}

${totalContext}

Your job:
1. Review the diff carefully
2. Use the full file contents and related files to trace root causes — not just surface issues
3. Identify bugs, security issues, logic errors, and bad patterns
4. For each issue, specify the exact file and line number

Return ONLY a JSON object with this structure (no markdown fences):
{
  "verdict": "PASS" or "FAIL",
  "summary": "2-3 sentence overall summary",
  "issues": [
    {
      "file": "path/to/file.js",
      "line": 42,
      "severity": "critical|high|medium|low",
      "title": "Short issue title",
      "detail": "Full explanation including root cause if cross-file"
    }
  ]
}
If no issues found, return an empty issues array and verdict PASS."""

                    writeFile file: 'prompt.txt', text: prompt

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=$CL_KEY
                            claude -p "$(cat prompt.txt)" --output-format json \
                                | jq -r '.result' \
                                | sed 's/```json//g' | sed 's/```//g' \
                                > review.json
                        '''
                    }

                    env.REVIEW_VERDICT = sh(script: "jq -r '.verdict // \"UNKNOWN\"' review.json", returnStdout: true).trim()
                }
            }
        }

        stage('Post to GitHub') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                    sh '''
                        # Build main PR comment with summary + issue table
                        VERDICT=$(jq -r '.verdict' review.json)
                        SUMMARY=$(jq -r '.summary' review.json)
                        ISSUE_COUNT=$(jq '.issues | length' review.json)

                        ICON="✅"
                        if [ "$VERDICT" = "FAIL" ]; then ICON="❌"; fi

                        # Build issues table rows
                        ISSUES_TABLE=$(jq -r '
                            .issues[] |
                            "| " + .severity + " | `" + .file + "` L" + (.line|tostring) + " | **" + .title + "** — " + .detail + " |"
                        ' review.json || echo "")

                        # Compose comment body
                        BODY=$(jq -n \
                            --arg icon "$ICON" \
                            --arg verdict "$VERDICT" \
                            --arg summary "$SUMMARY" \
                            --arg count "$ISSUE_COUNT" \
                            --arg table "$ISSUES_TABLE" \
                            '{body: ($icon + " ## Claude AI Review: " + $verdict + "\n\n**Summary:** " + $summary + "\n\n**Issues found:** " + $count + "\n\n| Severity | Location | Detail |\n|---|---|---|\n" + $table)}')

                        curl -sf -X POST \
                             -H "Authorization: token $TKN" \
                             -H "Content-Type: application/json" \
                             "https://api.github.com/repos/${REPO_PATH}/issues/${PR_NUMBER}/comments" \
                             -d "$BODY"

                        # Also post inline review comments for each issue (GitHub Review API)
                        jq -c '.issues[]' review.json | while read issue; do
                            FILE=$(echo $issue | jq -r '.file')
                            LINE=$(echo $issue | jq -r '.line')
                            TITLE=$(echo $issue | jq -r '.title')
                            DETAIL=$(echo $issue | jq -r '.detail')
                            SEV=$(echo $issue | jq -r '.severity')

                            INLINE_BODY=$(jq -n \
                                --arg body "**[$SEV] $TITLE**\n\n$DETAIL" \
                                --arg path "$FILE" \
                                --arg sha "${PR_HEAD_SHA}" \
                                --argjson line "$LINE" \
                                '{commit_id: $sha, path: $path, line: $line, side: "RIGHT", body: $body}')

                            curl -sf -X POST \
                                 -H "Authorization: token $TKN" \
                                 -H "Content-Type: application/json" \
                                 "https://api.github.com/repos/${REPO_PATH}/pulls/${PR_NUMBER}/comments" \
                                 -d "$INLINE_BODY" || true  # don't fail if line doesn't exist in diff
                        done
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                if (env.REPO_PATH && env.PR_HEAD_SHA) {
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
