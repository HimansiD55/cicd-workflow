
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
                    env.PR_NUMBER   = env.CHANGE_ID     ?: ''
                    env.BASE_BRANCH = env.CHANGE_TARGET ?: 'main'
                    env.HEAD_BRANCH = env.CHANGE_BRANCH ?: ''
                    env.PR_HEAD_SHA = env.GIT_COMMIT    ?: ''

                    echo "Repo: ${env.REPO_PATH} | PR #${env.PR_NUMBER} | ${env.HEAD_BRANCH} → ${env.BASE_BRANCH}"
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
                        diff = "No meaningful code changes detected (only lock files or build artifacts changed)."
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

                    echo "Diff size: ${diff.size()} bytes | Files:\n${changedFiles}"
                }
            }
        }

        stage('Claude AI Review') {
            steps {
                script {
                    def diff         = readFile('/tmp/pr_diff.txt').trim()
                    def changedFiles = readFile('/tmp/pr_files.txt').trim()
                    def prMeta       = readFile('/tmp/pr_meta.json').trim()

                    writeFile file: '/tmp/prompt.txt', text: """You are a senior software engineer performing a thorough code review of a GitHub Pull Request.

## PR Metadata
${prMeta}

## Changed Files
${changedFiles}

## Code Diff
<diff>
${diff}
</diff>

## Review Instructions

Analyze the diff carefully. For each issue include the file name and line number where possible.

1. Security Vulnerabilities - SQL injection, XSS, CSRF, insecure deserialization, improper auth, SSRF, OWASP Top 10
2. Secrets and Credential Leaks (CRITICAL) - Hardcoded API keys, tokens, passwords, private keys, base64 encoded secrets
3. Logic Bugs and Correctness - Off-by-one errors, null dereferences, race conditions, unhandled edge cases
4. Dependency Issues - Vulnerable package versions, missing pinning
5. Code Quality and Best Practices - DRY violations, missing input validation, poor naming, functions over 50 lines
6. Performance Issues - N+1 queries, missing pagination, inefficient algorithms

## Output Format

Respond with valid JSON only. No markdown, no preamble, nothing outside the JSON object:

{
  "summary": "One paragraph overall assessment.",
  "verdict": "PASS",
  "verdict_reason": "One sentence explaining the verdict.",
  "critical_issues": [],
  "warnings": [],
  "suggestions": [],
  "positive_notes": []
}

verdict must be PASS, WARN, or FAIL.
FAIL = any critical issue. WARN = warnings but no blockers. PASS = only suggestions or positives.
Each issue object must have: category, file, line (use ? if unknown), issue, recommendation"""

                    sh '''
python3 - <<'PYEOF'
import json

with open('/tmp/prompt.txt', 'r') as f:
    prompt_text = f.read()

payload = {
    "model": "claude-sonnet-4-5",
    "max_tokens": 4096,
    "system": "You are a code review assistant. Always respond with pure valid JSON only. Never use markdown code fences. Never add any text before or after the JSON object.",
    "messages": [{"role": "user", "content": prompt_text}]
}

with open('/tmp/claude_request.json', 'w') as f:
    json.dump(payload, f)

print("Request JSON written successfully")
PYEOF
'''

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CLAUDE_KEY')]) {
                        sh '''
                            HTTP_STATUS=$(curl -s -o /tmp/claude_response.json -w "%{http_code}" \
                                -X POST "https://api.anthropic.com/v1/messages" \
                                -H "x-api-key: ${CLAUDE_KEY}" \
                                -H "anthropic-version: 2023-06-01" \
                                -H "content-type: application/json" \
                                --data @/tmp/claude_request.json \
                                --max-time 90)

                            echo "HTTP Status: ${HTTP_STATUS}"
                            cat /tmp/claude_response.json

                            if [ "$HTTP_STATUS" != "200" ]; then
                                echo "ERROR: Anthropic API returned HTTP ${HTTP_STATUS}"
                                exit 1
                            fi
                        '''
                    }

                    def reviewJson = sh(
                        script: """
                            jq -r '.content[0].text // empty' /tmp/claude_response.json \
                            | sed 's/^```json//' \
                            | sed 's/^```//' \
                            | tr -d '\\r' \
                            | awk 'NF' \
                            > /tmp/review.json
                            cat /tmp/review.json
                        """,
                        returnStdout: false
                    )

                    def reviewContent = readFile('/tmp/review.json').trim()

                    if (!reviewContent) {
                        error("Claude API returned no content")
                    }

                    echo "Claude review received (${reviewContent.size()} bytes)"

                    sh "jq -r '.verdict // \"UNKNOWN\"' /tmp/review.json > /tmp/verdict.txt 2>/dev/null || echo 'UNKNOWN' > /tmp/verdict.txt"
                    env.REVIEW_VERDICT = readFile('/tmp/verdict.txt').trim()
                    echo "Verdict: ${env.REVIEW_VERDICT}"

                    if (!reviewJson) {
                        error("Claude API returned no content")
                    }

                    writeFile file: '/tmp/review.json', text: reviewJson
                    echo "Claude review received (${reviewJson.size()} bytes)"

                    sh "jq -r '.verdict // \"UNKNOWN\"' /tmp/review.json > /tmp/verdict.txt 2>/dev/null || echo 'UNKNOWN' > /tmp/verdict.txt"
                    env.REVIEW_VERDICT = readFile('/tmp/verdict.txt').trim()
                    echo "Verdict: ${env.REVIEW_VERDICT}"
                }
            }
        }

        stage('Post PR Comment') {
            steps {
                script {
                    def commentBody = sh(
                        script: """
sh '''
python3 - <<'PYEOF'
import json

with open('/tmp/prompt.txt', 'r') as f:
    prompt_text = f.read()

payload = {
    "model": "claude-sonnet-4-5",
    "max_tokens": 4096,
    "system": "You are a code review assistant. Always respond with pure valid JSON only. Never use markdown code fences. Never add any text before or after the JSON object.",
    "messages": [{"role": "user", "content": prompt_text}]
}

with open('/tmp/claude_request.json', 'w') as f:
    json.dump(payload, f)

print("Request JSON written successfully")
PYEOF
'''

verdict = data.get('verdict', 'UNKNOWN')
emoji_map = {'PASS': '\\u2705', 'WARN': '\\u26a0\\ufe0f', 'FAIL': '\\u274c'}
emoji = emoji_map.get(verdict, '\\U0001f50d')

lines = []
lines.append(f"## {emoji} Claude AI Code Review - {verdict}")
lines.append("")
lines.append(f"**{data.get('verdict_reason', '')}**")
lines.append("")
lines.append("### Summary")
lines.append(data.get('summary', ''))
lines.append("")

critical = data.get('critical_issues', [])
if critical:
    lines.append("---")
    lines.append("### Critical Issues (must fix before merge)")
    for i, issue in enumerate(critical, 1):
        lines.append(f"**{i}. [{issue.get('category','')}] {issue.get('file','')}:{issue.get('line','?')}**")
        lines.append(f"> {issue.get('issue','')}")
        lines.append(f"Fix: {issue.get('recommendation','')}")
        lines.append("")

warnings = data.get('warnings', [])
if warnings:
    lines.append("---")
    lines.append("### Warnings (should fix)")
    for i, w in enumerate(warnings, 1):
        lines.append(f"**{i}. [{w.get('category','')}] {w.get('file','')}:{w.get('line','?')}**")
        lines.append(f"> {w.get('issue','')}")
        lines.append(f"{w.get('recommendation','')}")
        lines.append("")

suggestions = data.get('suggestions', [])
if suggestions:
    lines.append("---")
    lines.append("### Suggestions (nice to have)")
    for s in suggestions:
        lines.append(f"- [{s.get('category','')}] {s.get('file','')}: {s.get('issue','')} - {s.get('recommendation','')}")
    lines.append("")

positives = data.get('positive_notes', [])
if positives:
    lines.append("---")
    lines.append("### What is done well")
    for p in positives:
        lines.append(f"- {p}")
    lines.append("")

lines.append("---")
lines.append("*Reviewed by Claude AI via Jenkins pipeline. Human review is still required.*")

print('\\n'.join(lines))
PYEOF
                        """,
                        returnStdout: true
                    ).trim()

                    writeFile file: '/tmp/comment_body.txt', text: commentBody

                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        sh '''
                            COMMENT_JSON=$(python3 -c "
import json
with open('/tmp/comment_body.txt') as f:
    body = f.read()
print(json.dumps({'body': body}))
")
                            curl -sf -X POST \
                                 -H "Authorization: token ${GH_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 -H "Content-Type: application/json" \
                                 "https://api.github.com/repos/${REPO_PATH}/issues/${PR_NUMBER}/comments" \
                                 -d "${COMMENT_JSON}" \
                            && echo "Comment posted successfully"
                        '''
                    }
                }
            }
        }

        stage('Set PR Status') {
            steps {
                script {
                    def verdict     = env.REVIEW_VERDICT ?: 'UNKNOWN'
                    def stateMap    = [PASS: 'success', WARN: 'success', FAIL: 'failure', UNKNOWN: 'error']
                    def descMap     = [
                        PASS:    'Claude AI review passed — no critical issues found',
                        WARN:    'Claude AI review: warnings found, review recommended',
                        FAIL:    'Claude AI review FAILED — critical issues must be resolved',
                        UNKNOWN: 'Claude AI review error — check Jenkins logs'
                    ]
                    def state       = stateMap[verdict] ?: 'error'
                    def description = descMap[verdict]  ?: 'Review status unknown'

                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        sh """
                            curl -sf -X POST \
                                 -H "Authorization: token ${GH_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 -H "Content-Type: application/json" \
                                 "https://api.github.com/repos/${env.REPO_PATH}/statuses/${env.PR_HEAD_SHA}" \
                                 -d '{
                                   "state":       "${state}",
                                   "target_url":  "${env.BUILD_URL}",
                                   "description": "${description}",
                                   "context":     "claude-ai-review"
                                 }' \
                            && echo "Status set to: ${state}"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f /tmp/prompt.txt /tmp/claude_request.json /tmp/claude_response.json /tmp/pr_diff.txt /tmp/pr_files.txt /tmp/pr_meta.json /tmp/review.json /tmp/comment_body.txt /tmp/verdict.txt || true'
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
