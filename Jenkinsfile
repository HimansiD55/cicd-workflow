// ─────────────────────────────────────────────────────────────────────────────
// Jenkinsfile — Claude AI PR Code Review Pipeline
// Triggers on pull_request events only (open / synchronize / reopen)
// ─────────────────────────────────────────────────────────────────────────────

pipeline {
    agent {
        label 'ec2-reviewer'
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    triggers {
        githubPullRequests(
            spec: '',
            triggerMode: 'HEAVY_HOOKS',
            events: [
                Open(),
                NonMergeable(),
                Commit()
            ],
            abortRunning: true
        )
    }

    environment {
        GITHUB_TOKEN   = credentials('github-token')
        CLAUDE_API_KEY = credentials('ANTHROPIC_API_KEY')

        REPO_OWNER  = "${env.CHANGE_FORK ?: env.ghprbGhRepository?.split('/')?.getAt(0) ?: ''}"
        REPO_NAME   = "${env.GIT_URL?.tokenize('/')?.last()?.replace('.git','') ?: ''}"
        PR_NUMBER   = "${env.CHANGE_ID ?: env.ghprbPullId ?: ''}"
        PR_HEAD_SHA = "${env.GIT_COMMIT ?: ''}"
        BASE_BRANCH = "${env.CHANGE_TARGET ?: 'main'}"
        HEAD_BRANCH = "${env.CHANGE_BRANCH ?: ''}"
    }

    stages {

        // ── Guard: skip non-PR builds (e.g. main branch direct build) ─────────
        stage('Validate PR Context') {
            steps {
                script {
                    if (!env.CHANGE_ID) {
                        echo "Not a PR build (branch: ${env.BRANCH_NAME}). Skipping pipeline."
                        currentBuild.result = 'NOT_BUILT'
                        error("Skipping non-PR build — CHANGE_ID is not set")
                    }
                    echo "PR #${env.CHANGE_ID} detected. Proceeding with review."
                }
            }
        }

        // ── Stage 1: Checkout ─────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                script {
                    echo "PR #${env.PR_NUMBER}: ${env.HEAD_BRANCH} → ${env.BASE_BRANCH}"

                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "origin/pr/${env.PR_NUMBER}/head"]],
                        userRemoteConfigs: [[
                            url: env.GIT_URL,
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

        // ── Stage 2: Get PR Diff ──────────────────────────────────────────────
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

                    def repoSegments = env.GIT_URL.tokenize('/')
                    def repoOwner = repoSegments[-2]
                    def repoName  = repoSegments[-1].replace('.git','')

                    def prMetadata = sh(
                        script: """
                            curl -sf -H "Authorization: token ${GITHUB_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 "https://api.github.com/repos/${repoOwner}/${repoName}/pulls/${env.PR_NUMBER}" \
                            | jq -r '{title: .title, body: (.body // "(no description)"), base: .base.ref, head: .head.ref}'
                        """,
                        returnStdout: true
                    ).trim()

                    writeFile file: '/tmp/pr_diff.txt',  text: diff
                    writeFile file: '/tmp/pr_files.txt', text: changedFiles
                    writeFile file: '/tmp/pr_meta.json', text: prMetadata

                    echo "Diff size: ${diff.size()} bytes | Changed files:\n${changedFiles}"
                }
            }
        }

        // ── Stage 3: Claude AI Review ─────────────────────────────────────────
        stage('Claude AI Review') {
            steps {
                script {
                    def diff         = readFile('/tmp/pr_diff.txt').trim()
                    def changedFiles = readFile('/tmp/pr_files.txt').trim()
                    def prMeta       = readFile('/tmp/pr_meta.json').trim()

                    def prompt = """You are a senior software engineer performing a thorough code review of a GitHub Pull Request.

## PR Metadata
${prMeta}

## Changed Files
${changedFiles}

## Code Diff
```diff
${diff}

## Review Instructions

Analyze the diff carefully and provide a structured review covering ALL of the following categories. For each issue found, include the **file name and line number** where possible.

### 1. 🔒 Security Vulnerabilities
- SQL injection, XSS, CSRF, insecure deserialization
- Improper authentication or authorization
- Insecure use of cryptography
- Server-side request forgery (SSRF)
- Any OWASP Top 10 issues

### 2. 🔑 Secrets & Credential Leaks (CRITICAL)
- Hardcoded API keys, tokens, passwords, private keys
- Database connection strings with credentials
- Any secrets that should be in environment variables
- Base64 encoded secrets

### 3. 🐛 Logic Bugs & Correctness
- Off-by-one errors, null/undefined dereferences
- Race conditions, deadlocks
- Incorrect error handling or swallowed exceptions
- Edge cases not handled

### 4. 📦 Dependency Issues
- Known vulnerable package versions (if visible in diff)
- Unnecessary or overly broad dependencies
- Missing dependency pinning

### 5. 🏗️ Code Quality & Best Practices
- DRY violations, overly complex functions
- Missing input validation
- Poor naming or unclear intent
- Missing or inadequate error handling
- Functions that are too long (>50 lines)

### 6. ⚡ Performance Issues
- N+1 query patterns
- Missing pagination on list endpoints
- Inefficient algorithms or data structures
- Unnecessary blocking operations

---

## Output Format

Respond in this EXACT format (valid JSON only, no markdown outside the JSON):

{
  "summary": "One paragraph overall assessment of this PR.",
  "verdict": "PASS",
  "verdict_reason": "One sentence explaining the verdict.",
  "critical_issues": [],
  "warnings": [],
  "suggestions": [],
  "positive_notes": []
}

verdict must be one of: PASS, WARN, FAIL
- FAIL  → any CRITICAL issue found (security, secrets, crashes)
- WARN  → warnings found but no critical blockers
- PASS  → only suggestions or positive notes

Be specific and actionable. Do not hallucinate line numbers — only include them if you can see them in the diff."""

                    def escapedPrompt = prompt
                        .replace('\\', '\\\\')
                        .replace('"',  '\\"')
                        .replace('\n', '\\n')
                        .replace('\r', '\\r')
                        .replace('\t', '\\t')

                    writeFile file: '/tmp/claude_request.json', text: """{
  "model": "claude-sonnet-4-5",
  "max_tokens": 4096,
  "messages": [
    {
      "role": "user",
      "content": "${escapedPrompt}"
    }
  ]
}"""

                    def response = sh(
                        script: """
                            curl -sf -X POST "https://api.anthropic.com/v1/messages" \\
                                 -H "x-api-key: ${CLAUDE_API_KEY}" \\
                                 -H "anthropic-version: 2023-06-01" \\
                                 -H "content-type: application/json" \\
                                 --data @/tmp/claude_request.json \\
                                 --max-time 90
                        """,
                        returnStdout: true
                    ).trim()

                    writeFile file: '/tmp/claude_response.json', text: response

                    def reviewJson = sh(
                        script: "jq -r '.content[0].text' /tmp/claude_response.json",
                        returnStdout: true
                    ).trim()

                    writeFile file: '/tmp/review.json', text: reviewJson
                    echo "Claude review received (${reviewJson.size()} bytes)"

                    // Write verdict to a file to avoid shell quoting issues
                    sh "jq -r '.verdict // \"UNKNOWN\"' /tmp/review.json > /tmp/verdict.txt 2>/dev/null || echo 'UNKNOWN' > /tmp/verdict.txt"
                    def verdict = readFile('/tmp/verdict.txt').trim()

                    env.REVIEW_VERDICT = verdict
                    echo "Verdict: ${verdict}"
                }
            }
        }

        // ── Stage 4: Post Comment on GitHub PR ────────────────────────────────
        stage('Post PR Comment') {
            steps {
                script {
                    def repoSegments = env.GIT_URL.tokenize('/')
                    def repoPath     = "${repoSegments[-2]}/${repoSegments[-1].replace('.git','')}"

                    def commentBody = sh(
                        script: """
python3 - <<'PYEOF'
import json, sys

with open('/tmp/review.json') as f:
    data = json.load(f)

verdict = data.get('verdict', 'UNKNOWN')
emoji_map = {'PASS': '\\u2705', 'WARN': '\\u26a0\\ufe0f', 'FAIL': '\\u274c'}
emoji = emoji_map.get(verdict, '\\U0001f50d')

lines = []
lines.append(f"## {emoji} Claude AI Code Review \\u2014 {verdict}")
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
        lines.append(f"**{i}. [{issue['category']}] {issue['file']}:{issue.get('line','?')}**")
        lines.append(f"> {issue['issue']}")
        lines.append(f"Fix: {issue['recommendation']}")
        lines.append("")

warnings = data.get('warnings', [])
if warnings:
    lines.append("---")
    lines.append("### Warnings (should fix)")
    for i, w in enumerate(warnings, 1):
        lines.append(f"**{i}. [{w['category']}] {w['file']}:{w.get('line','?')}**")
        lines.append(f"> {w['issue']}")
        lines.append(f"{w['recommendation']}")
        lines.append("")

suggestions = data.get('suggestions', [])
if suggestions:
    lines.append("---")
    lines.append("### Suggestions (nice to have)")
    for s in suggestions:
        lines.append(f"- [{s['category']}] {s['file']}: {s['issue']} - {s['recommendation']}")
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

                    // Write comment to file to avoid shell escaping issues
                    writeFile file: '/tmp/comment_body.txt', text: commentBody

                    sh """
                        COMMENT_JSON=\$(python3 -c "
import json, sys
with open('/tmp/comment_body.txt') as f:
    body = f.read()
print(json.dumps({'body': body}))
")
                        curl -sf -X POST \\
                             -H "Authorization: token ${GITHUB_TOKEN}" \\
                             -H "Accept: application/vnd.github.v3+json" \\
                             -H "Content-Type: application/json" \\
                             "https://api.github.com/repos/${repoPath}/issues/${env.PR_NUMBER}/comments" \\
                             -d "\$COMMENT_JSON" \\
                        && echo "Comment posted successfully"
                    """
                }
            }
        }

        // ── Stage 5: Set GitHub Commit Status ─────────────────────────────────
        stage('Set PR Status') {
            steps {
                script {
                    def repoSegments = env.GIT_URL.tokenize('/')
                    def repoPath     = "${repoSegments[-2]}/${repoSegments[-1].replace('.git','')}"
                    def verdict      = env.REVIEW_VERDICT ?: 'UNKNOWN'

                    def stateMap = [PASS: 'success', WARN: 'success', FAIL: 'failure', UNKNOWN: 'error']
                    def descMap  = [
                        PASS:    'Claude AI review passed — no critical issues found',
                        WARN:    'Claude AI review: warnings found, review recommended',
                        FAIL:    'Claude AI review FAILED — critical issues must be resolved',
                        UNKNOWN: 'Claude AI review error — check Jenkins logs'
                    ]

                    def state       = stateMap[verdict] ?: 'error'
                    def description = descMap[verdict]  ?: 'Review status unknown'
                    def buildUrl    = env.BUILD_URL ?: ''

                    sh """
                        curl -sf -X POST \\
                             -H "Authorization: token ${GITHUB_TOKEN}" \\
                             -H "Accept: application/vnd.github.v3+json" \\
                             -H "Content-Type: application/json" \\
                             "https://api.github.com/repos/${repoPath}/statuses/${env.PR_HEAD_SHA}" \\
                             -d '{
                               "state":       "${state}",
                               "target_url":  "${buildUrl}",
                               "description": "${description}",
                               "context":     "claude-ai-review"
                             }' \\
                        && echo "Status set to: ${state}"
                    """
                }
            }
        }
    }

    // ── Post actions ──────────────────────────────────────────────────────────
    post {
        always {
            sh 'rm -f /tmp/claude_request.json /tmp/claude_response.json /tmp/pr_diff.txt /tmp/comment_body.txt /tmp/verdict.txt'
            cleanWs()
        }

        success {
            script {
                def verdict = env.REVIEW_VERDICT ?: 'UNKNOWN'
                if (verdict == 'FAIL') {
                    currentBuild.result = 'UNSTABLE'
                }
            }
        }

        failure {
            script {
                def repoSegments = env.GIT_URL?.tokenize('/')
                def repoPath     = repoSegments ? "${repoSegments[-2]}/${repoSegments[-1]?.replace('.git','')}" : ''
                def sha          = env.PR_HEAD_SHA ?: ''
                if (repoPath && sha) {
                    sh """
                        curl -sf -X POST \\
                             -H "Authorization: token ${GITHUB_TOKEN}" \\
                             -H "Accept: application/vnd.github.v3+json" \\
                             "https://api.github.com/repos/${repoPath}/statuses/${sha}" \\
                             -d '{"state":"error","description":"Jenkins pipeline error","context":"claude-ai-review"}' || true
                    """
                }
            }
        }
    }
}
