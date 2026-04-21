pipeline {
    agent { label 'ec2-reviewer' }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        AGENT_CLASSES = 'security,logic,style,performance,documentation'
    }

    stages {

        stage('Validate PR Context') {
            steps {
                script {
                    if (!env.CHANGE_ID) {
                        currentBuild.result = 'NOT_BUILT'
                        error("Skipping non-PR build")
                    }
                    def urlSegments = (env.GIT_URL ?: scm.userRemoteConfigs[0].url ?: '').tokenize('/')
                    if (urlSegments.size() < 2) error("Cannot parse repo from URL")
                    env.REPO_PATH   = "${urlSegments[-2]}/${urlSegments[-1].replace('.git', '')}"
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
                    sh "git fetch origin ${env.BASE_BRANCH}:refs/remotes/origin/${env.BASE_BRANCH} --depth=50"
                }
            }
        }

        stage('Build Context Package') {
            steps {
                script {
                    // Write the Python JSON extractor once — reused by all agents and verifier
                    // Using writeFile avoids ALL Groovy/shell escaping issues with backticks and quotes
                    writeFile file: 'extract_json.py', text: '''
import sys
import re

text = sys.stdin.read()
mode = sys.argv[1] if len(sys.argv) > 1 else "object"

if mode == "array":
    # Try fenced array first
    match = re.search(r"```json\\s*(\\[.*?\\])\\s*```", text, re.DOTALL)
    if match:
        print(match.group(1))
        sys.exit(0)
    # Try raw array
    start = text.find("[")
    end   = text.rfind("]")
    if start != -1 and end != -1:
        print(text[start:end+1])
        sys.exit(0)
    print("[]")
else:
    # Try fenced object first
    match = re.search(r"```json\\s*(\\{.*?\\})\\s*```", text, re.DOTALL)
    if match:
        print(match.group(1))
        sys.exit(0)
    # Try raw object
    start = text.find("{")
    end   = text.rfind("}")
    if start != -1 and end != -1:
        print(text[start:end+1])
        sys.exit(0)
    print(text.strip())
'''

                    def diff = sh(
                        script: "git diff origin/${env.BASE_BRANCH}...HEAD -- ':!*.lock' ':!package-lock.json' ':!yarn.lock'",
                        returnStdout: true
                    ).trim()
                    writeFile file: 'pr_diff.txt', text: diff ?: "No changes"

                    def changedFiles = sh(
                        script: "git diff --name-only origin/${env.BASE_BRANCH}...HEAD -- ':!*.lock' ':!package-lock.json' ':!yarn.lock'",
                        returnStdout: true
                    ).trim().split('\n').findAll { it }
                    env.CHANGED_FILES_LIST = changedFiles.join(', ')

                    def contextBuilder = new StringBuilder("=== FULL CONTENTS OF CHANGED FILES ===\n\n")
                    changedFiles.each { filePath ->
                        if (fileExists(filePath)) {
                            def fileContent = readFile(filePath)
                            contextBuilder.append("--- FILE: ${filePath} ---\n${fileContent.size() > 30000 ? fileContent.take(30000) + '\n...[truncated]' : fileContent}\n\n")
                        }
                    }
                    writeFile file: 'full_context.txt', text: contextBuilder.toString()

                    // Read once here — passed into runAgent() to avoid 6x repeated disk reads
                    env.SHARED_DIFF    = readFile('pr_diff.txt')
                    env.SHARED_CONTEXT = readFile('full_context.txt')
                }
            }
        }

        // ── PHASE 1: Parallel specialized agents ─────────────────────────────
        stage('Parallel Agent Review') {
            parallel {

                stage('Agent: Security') {
                    steps {
                        script { runAgent('security',
                            'Focus ONLY on security issues: injection flaws, insecure deserialization, hardcoded secrets, improper auth, dangerous eval/exec usage, path traversal, unvalidated input, XSS, CSRF, and dependency vulnerabilities.',
                            env.SHARED_DIFF, env.SHARED_CONTEXT
                        )}
                    }
                }

                stage('Agent: Logic') {
                    steps {
                        script { runAgent('logic',
                            'Focus ONLY on logic and correctness: off-by-one errors, null pointer risks, race conditions, incorrect algorithms, unreachable code, infinite loops, wrong operator precedence, missing error handling, and incorrect type coercion.',
                            env.SHARED_DIFF, env.SHARED_CONTEXT
                        )}
                    }
                }

                stage('Agent: Style') {
                    steps {
                        script { runAgent('style',
                            'Focus ONLY on code style and maintainability: naming conventions, function length, cyclomatic complexity, dead code, duplicated logic, poor abstractions, missing comments on non-obvious code, and test coverage gaps.',
                            env.SHARED_DIFF, env.SHARED_CONTEXT
                        )}
                    }
                }

                stage('Agent: Performance') {
                    steps {
                        script { runAgent('performance',
                            'Focus ONLY on performance: N+1 queries, missing indexes hinted by ORM calls, unnecessary allocations in hot paths, blocking I/O on async threads, inefficient data structures, and missing caching opportunities.',
                            env.SHARED_DIFF, env.SHARED_CONTEXT
                        )}
                    }
                }

                stage('Agent: Documentation') {
                    steps {
                        script { runAgent('documentation',
                            'Focus ONLY on documentation quality: missing or misleading docstrings, outdated comments that contradict the code, undocumented public APIs, missing changelog entries, and README gaps for new env vars or config keys.',
                            env.SHARED_DIFF, env.SHARED_CONTEXT
                        )}
                    }
                }
            }
        }

        // ── PHASE 2: Verification + dedup ─────────────────────────────────────
        stage('Verification + Dedup') {
            steps {
                script {
                    def agentClasses = env.AGENT_CLASSES.tokenize(',')

                    // Warn if any agent silently fell back to empty results
                    def fallbackAgents = agentClasses.findAll { cls ->
                        fileExists("agent_${cls}_fallback.flag")
                    }
                    if (fallbackAgents) {
                        unstable("WARNING: agents ${fallbackAgents.join(', ')} failed and returned empty results — review may be incomplete")
                    }

                    def candidates = new StringBuilder("=== RAW CANDIDATES FROM ALL AGENTS ===\n\n")
                    agentClasses.each { cls ->
                        def agentFile = "agent_${cls}.json"
                        if (fileExists(agentFile)) {
                            candidates.append("--- ${cls.toUpperCase()} AGENT ---\n")
                            candidates.append(readFile(agentFile))
                            candidates.append("\n\n")
                        }
                    }
                    writeFile file: 'candidates.txt', text: candidates.toString()

                    writeFile file: 'verify_prompt.txt', text: """You are a senior verification engineer. You have received candidate issues from ${agentClasses.size()} specialized review agents.

Your job:
1. Read each candidate issue carefully.
2. Cross-check it against the actual code shown in the full context below.
3. DISCARD any issue that is a false positive.
4. DEDUPLICATE issues flagged by multiple agents.
5. RANK remaining issues by severity: critical then high then medium then low.
6. Emit a final consolidated verdict.

CHANGED FILES: ${env.CHANGED_FILES_LIST}

=== GIT DIFF ===
${env.SHARED_DIFF}

${env.SHARED_CONTEXT}

=== CANDIDATE ISSUES FROM ALL AGENTS ===
${readFile('candidates.txt')}

IMPORTANT: Your entire response must be the raw JSON object only.
No explanation, no prose, no markdown fences. Start with { and end with }.

{
  "verdict": "PASS or FAIL",
  "summary": "2-3 sentence summary of the overall PR quality",
  "agent_stats": {"security":0,"logic":0,"style":0,"performance":0,"documentation":0},
  "issues": [
    {
      "file": "...",
      "line": 1,
      "severity": "critical|high|medium|low",
      "category": "security|logic|style|performance|documentation",
      "title": "...",
      "detail": "...",
      "false_positive_reason": null
    }
  ]
}

Verdict is FAIL only if at least one critical or high severity issue remains after verification.
If no real issues found, return empty issues array with verdict PASS."""

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=$CL_KEY
                            claude -p "$(cat verify_prompt.txt)" --output-format json > verify_raw.json 2>&1 || true
                            jq -r ".result // empty" verify_raw.json | python3 extract_json.py object > review.json
                            jq empty review.json || (echo "Verifier returned invalid JSON:" && cat verify_raw.json && exit 1)
                        '''
                    }

                    env.REVIEW_VERDICT = sh(
                        script: 'jq -r \'.verdict // "UNKNOWN"\' review.json',
                        returnStdout: true
                    ).trim()

                    // Sanitize — whitelist before any value touches a shell string
                    def rawVerdict    = env.REVIEW_VERDICT ?: 'UNKNOWN'
                    env.SAFE_VERDICT  = (rawVerdict in ['PASS', 'FAIL']) ? rawVerdict : 'UNKNOWN'
                    env.COMMIT_STATE  = (env.SAFE_VERDICT == 'PASS') ? 'success'
                                      : (env.SAFE_VERDICT == 'FAIL') ? 'failure'
                                      : 'error'
                }
            }
        }

        stage('Post to GitHub') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                    sh '''
                        VERDICT=$(jq -r ".verdict" review.json)
                        SUMMARY=$(jq -r ".summary" review.json)
                        ISSUE_COUNT=$(jq ".issues | length" review.json)
                        ICON="✅"; [ "$VERDICT" = "FAIL" ] && ICON="❌"

                        STATS=$(jq -r "
                            .agent_stats // {} |
                            to_entries |
                            map(.key + \": \" + (.value|tostring)) |
                            join(\"  |  \")
                        " review.json 2>/dev/null || echo "")

                        ISSUES_TABLE=$(jq -r "
                            .issues[] |
                            \"| \" + .severity +
                            \" | \" + .file + \" L\" + (.line|tostring) +
                            \" | \" + .category +
                            \" | \" + .title + \" — \" + .detail + \" |\"
                        " review.json 2>/dev/null || echo "")

                        jq -n --arg b "${ICON} ## Claude AI Review (multi-agent): ${VERDICT}

**Summary:** ${SUMMARY}

**Issues found:** ${ISSUE_COUNT}  |  ${STATS}

| Severity | Location | Category | Detail |
|---|---|---|---|
${ISSUES_TABLE}

> Reviewed by ${AGENT_CLASSES} agents in parallel, verified and deduplicated." \
                            "{body: \$b}" > pr_comment.json

                        curl -sf -X POST \
                             -H "Authorization: token $TKN" \
                             -H "Content-Type: application/json" \
                             "https://api.github.com/repos/${REPO_PATH}/issues/${PR_NUMBER}/comments" \
                             -d @pr_comment.json

                        # Batch-extract all issue fields in one jq call, POST in parallel
                        jq -r ".issues[] | [.severity, .category, .title, .detail, .file, (.line|tostring)] | @tsv" \
                            review.json > /tmp/issues.tsv 2>/dev/null || true

                        while IFS=$(printf "\\t") read -r severity category title detail filePath lineNum; do
                            echo "$lineNum" | grep -qE "^[0-9]+$" || continue
                            jq -n \
                                --arg body "[${severity}] [${category}] ${title}

${detail}" \
                                --arg path "$filePath" \
                                --arg sha  "$PR_HEAD_SHA" \
                                --argjson line "$lineNum" \
                                "{commit_id:\$sha, path:\$path, line:\$line, side:\"RIGHT\", body:\$body}" \
                            | curl -sf -X POST \
                                   -H "Authorization: token $TKN" \
                                   -H "Content-Type: application/json" \
                                   "https://api.github.com/repos/${REPO_PATH}/pulls/${PR_NUMBER}/comments" \
                                   -d @- &
                        done < /tmp/issues.tsv
                        wait
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                if (env.PR_HEAD_SHA) {
                    // Only whitelisted values reach here — no AI-derived string in shell
                    def safeVerdict = env.SAFE_VERDICT ?: 'UNKNOWN'
                    def commitState = env.COMMIT_STATE ?: 'error'
                    withCredentials([string(credentialsId: 'github-token', variable: 'TKN')]) {
                        sh """
                            jq -n \
                                --arg state "${commitState}" \
                                --arg desc  "Multi-agent verdict: ${safeVerdict}" \
                                '{"state":\$state,"context":"claude-ai-review","description":\$desc}' \
                            | curl -sf -X POST \
                                   -H "Authorization: token \$TKN" \
                                   -H "Content-Type: application/json" \
                                   "https://api.github.com/repos/${env.REPO_PATH}/statuses/${env.PR_HEAD_SHA}" \
                                   -d @- || true
                        """
                    }
                }
            }
            cleanWs()
        }
    }
}

// ── Shared helper ─────────────────────────────────────────────────────────────
// Runs one specialized agent and saves its raw findings to agent_<class>.json.
//
// Parameters:
//   agentClass — one of: security, logic, style, performance, documentation
//   mandate    — focused instruction string describing the agent's issue class
//   diff       — pre-read contents of pr_diff.txt (avoids repeated disk reads)
//   context    — pre-read contents of full_context.txt (avoids repeated disk reads)
//
// Preconditions:
//   - env.CHANGED_FILES_LIST must be set
//   - extract_json.py must exist in workspace (written by Build Context Package)
//   - ANTHROPIC_API_KEY Jenkins credential must exist
//
// Side-effects (files written to workspace):
//   - prompt_<agentClass>.txt        — full prompt sent to Claude
//   - raw_<agentClass>.json          — raw claude CLI output
//   - agent_<agentClass>.json        — extracted JSON array of issues ([] on failure)
//   - agent_<agentClass>_fallback.flag — created only when API failed and [] was used
// ─────────────────────────────────────────────────────────────────────────────
def runAgent(String agentClass, String mandate, String diff, String context) {
    writeFile file: "prompt_${agentClass}.txt", text: """You are a specialized code review agent. Your mandate:

${mandate}

You must ONLY report issues within your mandate. Be strict — only flag real, confirmed issues.

CHANGED FILES: ${env.CHANGED_FILES_LIST}

=== GIT DIFF ===
${diff}

${context}

IMPORTANT: Your entire response must be a raw JSON array only.
No explanation, no prose, no markdown fences. Start with [ and end with ].

[
  {
    "file": "...",
    "line": 1,
    "severity": "critical|high|medium|low",
    "title": "Short title",
    "detail": "Detailed explanation and suggested fix"
  }
]"""

    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
        sh """
            export ANTHROPIC_API_KEY=\$CL_KEY
            claude -p "\$(cat prompt_${agentClass}.txt)" --output-format json > raw_${agentClass}.json 2>&1 || true

            jq -r ".result // \"[]\"" raw_${agentClass}.json | python3 extract_json.py array > agent_${agentClass}.json

            if ! jq empty agent_${agentClass}.json 2>/dev/null; then
                echo "WARNING: ${agentClass} agent returned invalid JSON — falling back to []"
                echo "[]" > agent_${agentClass}.json
                touch agent_${agentClass}_fallback.flag
            fi
        """
    }
}
