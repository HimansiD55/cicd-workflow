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
                    def segs   = gitUrl.tokenize('/')
                    if (segs.size() < 2) error("Cannot parse repo URL: ${gitUrl}")
                    env.REPO_OWNER  = segs[-2]
                    env.REPO_NAME   = segs[-1].replace('.git', '')
                    env.REPO_PATH   = "${env.REPO_OWNER}/${env.REPO_NAME}"
                    env.PR_NUMBER   = env.CHANGE_ID
                    env.BASE_BRANCH = env.CHANGE_TARGET ?: 'main'
                    echo "Repo: ${env.REPO_PATH} | PR #${env.PR_NUMBER} | target: ${env.BASE_BRANCH}"
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
                        extensions: [[$class: 'CloneOption', depth: 0, noTags: true, shallow: false]]
                    ])
                    env.PR_HEAD_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    echo "HEAD: ${env.PR_HEAD_SHA}"
                }
            }
        }

        stage('Collect Diff and Full File Context') {
            steps {
                script {
                    sh "git fetch origin ${env.BASE_BRANCH}:refs/remotes/origin/${env.BASE_BRANCH} --depth=50"

                    // Get the diff
                    def diff = sh(
                        script: """
                            git diff origin/${env.BASE_BRANCH}...HEAD \
                                -- ':!*.lock' ':!package-lock.json' ':!yarn.lock' \
                                   ':!*.min.js' ':!*.min.css' ':!dist/' ':!build/' \
                                2>/dev/null | head -c 60000
                        """,
                        returnStdout: true
                    ).trim() ?: "No meaningful changes detected."

                    // Get list of changed files with line numbers
                    def changedFiles = sh(
                        script: "git diff --name-only origin/${env.BASE_BRANCH}...HEAD 2>/dev/null | head -30",
                        returnStdout: true
                    ).trim()

                    // Get full content of each changed file for root cause analysis
                    def fullContext = ""
                    if (changedFiles) {
                        changedFiles.split('\n').each { file ->
                            file = file.trim()
                            if (file && fileExists(file)) {
                                def content = sh(
                                    script: "cat '${file}' 2>/dev/null | head -c 8000",
                                    returnStdout: true
                                ).trim()
                                fullContext += "\n\n=== FULL FILE: ${file} ===\n${content}"
                            }
                        }
                    }

                    // Get PR metadata from GitHub
                    def prMeta = sh(
                        script: """
                            curl -sf \
                                 -H "Authorization: token ${GITHUB_TOKEN}" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 "https://api.github.com/repos/${env.REPO_PATH}/pulls/${env.PR_NUMBER}" \
                            | jq -r '{title: .title, body: (.body // "(no description)"), base: .base.ref, head: .head.ref}'
                        """,
                        returnStdout: true
                    ).trim()

                    writeFile file: '/tmp/pr_diff.txt',     text: diff
                    writeFile file: '/tmp/pr_files.txt',    text: changedFiles
                    writeFile file: '/tmp/pr_context.txt',  text: fullContext
                    writeFile file: '/tmp/pr_meta.json',    text: prMeta

                    echo "Diff: ${diff.size()} bytes | Files changed:\n${changedFiles}"
                }
            }
        }

        stage('Claude AI Review') {
            steps {
                script {
                    def diff        = readFile('/tmp/pr_diff.txt').trim()
                    def files       = readFile('/tmp/pr_files.txt').trim()
                    def fullContext = readFile('/tmp/pr_context.txt').trim()
                    def prMeta      = readFile('/tmp/pr_meta.json').trim()

                    // Write prompt to file
                    writeFile file: '/tmp/prompt.txt', text: """You are an expert code reviewer with deep knowledge of security, performance, and software engineering best practices.

## PR Metadata
${prMeta}

## Changed Files
${files}

## Code Diff (what changed)
<diff>
${diff}
</diff>

## Full File Contents (for root cause analysis across files)
<full_context>
${fullContext}
</full_context>

## Your Task

Perform a thorough code review. Use the full file contents to trace root causes across files — do not limit analysis to only the changed lines.

For each issue found, identify the EXACT file and line number from the diff or full context.

### Categories to check:
1. Security - SQL injection, XSS, CSRF, hardcoded secrets, insecure auth, SSRF
2. Secrets - API keys, passwords, tokens hardcoded anywhere in the files
3. Logic Bugs - null dereferences, race conditions, wrong error handling, edge cases
4. Dependencies - vulnerable versions, missing pinning
5. Code Quality - DRY violations, poor naming, functions over 50 lines, missing validation
6. Performance - N+1 queries, missing pagination, blocking operations

## Output Format

Return ONLY a valid JSON object. No markdown. No explanation outside the JSON:

{
  "summary": "Overall assessment in 2-3 sentences.",
  "verdict": "PASS",
  "verdict_reason": "One sentence explaining verdict.",
  "critical_issues": [
    {
      "file": "src/app.js",
      "line": 42,
      "category": "Security",
      "issue": "Clear description of the problem",
      "recommendation": "Specific fix to apply"
    }
  ],
  "warnings": [
    {
      "file": "src/app.js",
      "line": 10,
      "category": "Code Quality",
      "issue": "Description",
      "recommendation": "Fix"
    }
  ],
  "suggestions": [
    {
      "file": "src/app.js",
      "line": "?",
      "category": "Performance",
      "issue": "Description",
      "recommendation": "Fix"
    }
  ],
  "positive_notes": ["What was done well"]
}

verdict must be PASS, WARN, or FAIL.
FAIL = critical_issues is not empty.
WARN = warnings exist but no critical issues.
PASS = only suggestions or positive notes."""

                    // Build request JSON safely using Python
                    sh '''
python3 - <<'PYEOF'
import json

with open('/tmp/prompt.txt', 'r') as f:
    prompt_text = f.read()

payload = {
    "model": "claude-sonnet-4-5",
    "max_tokens": 8096,
    "system": "You are an expert code reviewer. Always respond with pure valid JSON only. Never use markdown fences. Never add text before or after the JSON object.",
    "messages": [{"role": "user", "content": prompt_text}]
}

with open('/tmp/claude_request.json', 'w') as f:
    json.dump(payload, f)

print("Request JSON written successfully")
PYEOF
'''

                    // Call Claude API
                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CLAUDE_KEY')]) {
                        sh '''
                            HTTP_STATUS=$(curl -s -o /tmp/claude_response.json -w "%{http_code}" \
                                -X POST "https://api.anthropic.com/v1/messages" \
                                -H "x-api-key: ${CLAUDE_KEY}" \
                                -H "anthropic-version: 2023-06-01" \
                                -H "content-type: application/json" \
                                --data @/tmp/claude_request.json \
                                --max-time 120)

                            echo "HTTP Status: ${HTTP_STATUS}"
                            if [ "$HTTP_STATUS" != "200" ]; then
                                cat /tmp/claude_response.json
                                echo "ERROR: Anthropic API returned HTTP ${HTTP_STATUS}"
                                exit 1
                            fi
                        '''
                    }

                    // Extract and clean the JSON review
                    sh '''
                        jq -r '.content[0].text // empty' /tmp/claude_response.json \
                            | sed 's/^```json[[:space:]]*//' \
                            | sed 's/^```[[:space:]]*//' \
                            | sed 's/[[:space:]]*```$//' \
                            | awk 'NF' \
                            > /tmp/review.json
                    '''

                    def reviewContent = readFile('/tmp/review.json').trim()
                    if (!reviewContent) error("Claude returned no content")

                    echo "Review received (${reviewContent.size()} bytes)"

                    sh "jq -r '.verdict // \"UNKNOWN\"' /tmp/review.json > /tmp/verdict.txt 2>/dev/null || echo 'UNKNOWN' > /tmp/verdict.txt"
                    env.REVIEW_VERDICT = readFile('/tmp/verdict.txt').trim()
                    echo "Verdict: ${env.REVIEW_VERDICT}"
                }
            }
        }

        stage('Post Inline Comments + PR Summary') {
            steps {
                script {
                    // Post inline review comments on specific lines using GitHub Review API
                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        sh '''
python3 - <<'PYEOF'
import json, subprocess, os

with open('/tmp/review.json') as f:
    review = json.load(f)

repo_path = os.environ['REPO_PATH']
pr_number = os.environ['PR_NUMBER']
head_sha  = os.environ['PR_HEAD_SHA']
gh_token  = os.environ['GH_TOKEN']

verdict = review.get('verdict', 'UNKNOWN')
emoji_map = {'PASS': '✅', 'WARN': '⚠️', 'FAIL': '❌'}
emoji = emoji_map.get(verdict, '🔍')

# ── Build PR summary body ──────────────────────────────────────────
lines = []
lines.append(f"## {emoji} Claude AI Code Review — {verdict}")
lines.append("")
lines.append(f"**{review.get('verdict_reason', '')}**")
lines.append("")
lines.append("### Summary")
lines.append(review.get('summary', ''))
lines.append("")

critical = review.get('critical_issues', [])
if critical:
    lines.append("---")
    lines.append("### 🔴 Critical Issues (must fix before merge)")
    for i, issue in enumerate(critical, 1):
        lines.append(f"**{i}. [{issue.get('category','')}] `{issue.get('file','')}` line {issue.get('line','?')}**")
        lines.append(f"> {issue.get('issue','')}")
        lines.append(f"**Fix:** {issue.get('recommendation','')}")
        lines.append("")

warnings = review.get('warnings', [])
if warnings:
    lines.append("---")
    lines.append("### ⚠️ Warnings (should fix)")
    for i, w in enumerate(warnings, 1):
        lines.append(f"**{i}. [{w.get('category','')}] `{w.get('file','')}` line {w.get('line','?')}**")
        lines.append(f"> {w.get('issue','')}")
        lines.append(f"**Fix:** {w.get('recommendation','')}")
        lines.append("")

suggestions = review.get('suggestions', [])
if suggestions:
    lines.append("---")
    lines.append("### 💡 Suggestions (nice to have)")
    for s in suggestions:
        lines.append(f"- [`{s.get('file','')}`] {s.get('issue','')} — {s.get('recommendation','')}")
    lines.append("")

positives = review.get('positive_notes', [])
if positives:
    lines.append("---")
    lines.append("### ✨ What is done well")
    for p in positives:
        lines.append(f"- {p}")
    lines.append("")

lines.append("---")
lines.append("*Reviewed by Claude AI via Jenkins. Human review is still required.*")

summary_body = '\n'.join(lines)

# ── Build inline review comments for critical issues and warnings ──
inline_comments = []
all_issues = critical + warnings
for issue in all_issues:
    line_val = issue.get('line', '?')
    file_val = issue.get('file', '')
    # Only add inline comment if we have a real file and line number
    try:
        line_num = int(str(line_val))
        if file_val and line_num > 0:
            body = f"**[{issue.get('category','')}]** {issue.get('issue','')}\n\n**Recommendation:** {issue.get('recommendation','')}"
            inline_comments.append({
                "path": file_val,
                "line": line_num,
                "side": "RIGHT",
                "body": body
            })
    except (ValueError, TypeError):
        pass  # skip issues without a valid line number

# ── Post GitHub Pull Request Review (inline + summary) ────────────
review_payload = {
    "commit_id": head_sha,
    "body": summary_body,
    "event": "COMMENT",
    "comments": inline_comments
}

payload_str = json.dumps(review_payload)
with open('/tmp/review_payload.json', 'w') as f:
    f.write(payload_str)

result = subprocess.run([
    'curl', '-sf', '-X', 'POST',
    '-H', f'Authorization: token {gh_token}',
    '-H', 'Accept: application/vnd.github.v3+json',
    '-H', 'Content-Type: application/json',
    f'https://api.github.com/repos/{repo_path}/pulls/{pr_number}/reviews',
    '-d', f'@/tmp/review_payload.json'
], capture_output=True, text=True)

if result.returncode == 0:
    print(f"Review posted successfully with {len(inline_comments)} inline comment(s)")
else:
    print(f"ERROR posting review: {result.stderr}")
    # Fallback: post as plain comment
    fallback = json.dumps({"body": summary_body})
    subprocess.run([
        'curl', '-sf', '-X', 'POST',
        '-H', f'Authorization: token {gh_token}',
        '-H', 'Content-Type: application/json',
        f'https://api.github.com/repos/{repo_path}/issues/{pr_number}/comments',
        '-d', fallback
    ])
    print("Fallback comment posted")

PYEOF
'''
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                if (env.REPO_PATH && env.PR_HEAD_SHA) {
                    def verdict     = env.REVIEW_VERDICT ?: 'UNKNOWN'
                    def stateMap    = [PASS: 'success', WARN: 'success', FAIL: 'failure', UNKNOWN: 'error']
                    def state       = stateMap[verdict] ?: 'error'
                    def descMap     = [
                        PASS:    'Claude AI review passed',
                        WARN:    'Claude AI review: warnings found',
                        FAIL:    'Claude AI review FAILED — fix critical issues',
                        UNKNOWN: 'Claude AI review error'
                    ]
                    def description = descMap[verdict] ?: 'Review error'

                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        sh """
                            curl -sf -X POST \
                                 -H "Authorization: token \$GH_TOKEN" \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 -H "Content-Type: application/json" \
                                 "https://api.github.com/repos/${env.REPO_PATH}/statuses/${env.PR_HEAD_SHA}" \
                                 -d '{"state":"${state}","context":"claude-ai-review","description":"${description}","target_url":"${env.BUILD_URL}"}' || true
                        """
                    }
                }
            }
            sh 'rm -f /tmp/prompt.txt /tmp/claude_request.json /tmp/claude_response.json /tmp/pr_diff.txt /tmp/pr_files.txt /tmp/pr_context.txt /tmp/pr_meta.json /tmp/review.json /tmp/review_payload.json /tmp/verdict.txt || true'
            cleanWs()
        }
    }
}
