pipeline {
    agent { label 'ec2-reviewer' }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        GITHUB_TOKEN   = credentials('github-token')
        CLAUDE_API_KEY = credentials('ANTHROPIC_API_KEY')
        // Agent specializations — each targets a distinct issue class
        AGENT_CLASSES  = 'security,logic,style,performance,documentation'
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

        // ── PHASE 1: Parallel specialized agents ─────────────────────────────
        // Each agent gets the same diff+context but a narrow, focused mandate.
        // Isolated context windows = each agent is unbiased by another's findings.
        stage('Parallel Agent Review') {
            parallel {

                stage('Agent: Security') {
                    steps {
                        script { runAgent('security',
                            """Focus ONLY on security issues: injection flaws, insecure deserialization,
hardcoded secrets, improper auth, dangerous eval/exec usage, path traversal,
unvalidated input, XSS, CSRF, and dependency vulnerabilities."""
                        )}
                    }
                }

                stage('Agent: Logic') {
                    steps {
                        script { runAgent('logic',
                            """Focus ONLY on logic and correctness: off-by-one errors, null pointer risks,
race conditions, incorrect algorithms, unreachable code, infinite loops,
wrong operator precedence, missing error handling, and incorrect type coercion."""
                        )}
                    }
                }

                stage('Agent: Style') {
                    steps {
                        script { runAgent('style',
                            """Focus ONLY on code style and maintainability: naming conventions,
function length, cyclomatic complexity, dead code, duplicated logic,
poor abstractions, missing comments on non-obvious code, and test coverage gaps."""
                        )}
                    }
                }

                stage('Agent: Performance') {
                    steps {
                        script { runAgent('performance',
                            """Focus ONLY on performance: N+1 queries, missing indexes hinted by ORM calls,
unnecessary allocations in hot paths, blocking I/O on async threads,
inefficient data structures, and missing caching opportunities."""
                        )}
                    }
                }

                stage('Agent: Documentation') {
                    steps {
                        script { runAgent('documentation',
                            """Focus ONLY on documentation quality: missing or misleading docstrings,
outdated comments that contradict the code, undocumented public APIs,
missing changelog entries, and README gaps for new env vars or config keys."""
                        )}
                    }
                }
            }
        }

        // ── PHASE 2: Verification agent ───────────────────────────────────────
        // Reads all raw agent findings and cross-checks each candidate issue
        // against the actual code to filter false positives before ranking.
        stage('Verification + Dedup') {
            steps {
                script {
                    // Merge all agent outputs into one candidates file
                    def candidates = new StringBuilder("=== RAW CANDIDATES FROM ALL AGENTS ===\n\n")
                    ['security','logic','style','performance','documentation'].each { cls ->
                        def f = "agent_${cls}.json"
                        if (fileExists(f)) {
                            candidates.append("--- ${cls.toUpperCase()} AGENT ---\n")
                            candidates.append(readFile(f))
                            candidates.append("\n\n")
                        }
                    }
                    writeFile file: 'candidates.txt', text: candidates.toString()

                    def verifyPrompt = """You are a senior verification engineer. You have received candidate issues
from ${env.AGENT_CLASSES.split(',').size()} specialized review agents.

Your job:
1. Read each candidate issue carefully.
2. Cross-check it against the actual code shown in the full context below.
3. DISCARD any issue that is a false positive (the code already handles it,
   the concern doesn't apply to this language/framework, or the line cited
   doesn't exist).
4. DEDUPLICATE issues that were flagged by multiple agents.
5. RANK remaining issues by severity: critical → high → medium → low.
6. Emit a final consolidated verdict.

CHANGED FILES: ${env.CHANGED_FILES_LIST}

=== GIT DIFF ===
${readFile('pr_diff.txt')}

${readFile('full_context.txt')}

=== CANDIDATE ISSUES FROM ALL AGENTS ===
${readFile('candidates.txt')}

Return ONLY a JSON object, no markdown fences:
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
Verdict is FAIL only if there is at least one critical or high severity issue remaining after verification.
If no real issues found, return empty issues array with verdict PASS."""

                    writeFile file: 'verify_prompt.txt', text: verifyPrompt

                    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
                        sh '''
                            export ANTHROPIC_API_KEY=$CL_KEY
                            claude -p "$(cat verify_prompt.txt)" --output-format json > verify_raw.json 2>&1 || true
                            jq -r '.result // empty' verify_raw.json \
                                | sed 's/^```json//; s/^```//; s/```$//' > review.json
                            jq empty review.json || (echo "Verifier returned invalid JSON:" && cat verify_raw.json && exit 1)
                        '''
                    }

                    env.REVIEW_VERDICT = sh(
                        script: 'jq -r \'.verdict // "UNKNOWN"\' review.json',
                        returnStdout: true
                    ).trim()
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

                        # Build per-category stats line
                        STATS=$(jq -r '
                            .agent_stats // {} |
                            to_entries |
                            map("**" + .key + "**: " + (.value|tostring)) |
                            join("  ·  ")
                        ' review.json 2>/dev/null || echo "")

                        ISSUES_TABLE=$(jq -r '
                            .issues[] |
                            "| " + .severity +
                            " | `" + .file + "` L" + (.line|tostring) +
                            " | " + .category +
                            " | **" + .title + "** — " + .detail + " |"
                        ' review.json 2>/dev/null || echo "")

                        jq -n --arg b "$ICON ## Claude AI Review (multi-agent): $VERDICT

**Summary:** $SUMMARY

**Issues found:** $ISSUE_COUNT  |  $STATS

| Severity | Location | Category | Detail |
|---|---|---|---|
$ISSUES_TABLE

> _Reviewed by ${AGENT_CLASSES} agents in parallel, verified and deduplicated._" \
                            '{body: $b}' > pr_comment.json

                        curl -sf -X POST \
                             -H "Authorization: token $TKN" \
                             -H "Content-Type: application/json" \
                             "https://api.github.com/repos/${REPO_PATH}/issues/${PR_NUMBER}/comments" \
                             -d @pr_comment.json

                        # Inline comments per verified issue
                        i=0
                        while [ $i -lt $ISSUE_COUNT ]; do
                            jq -c ".issues[$i]" review.json > /tmp/issue.json
                            LINE=$(jq -r '.line' /tmp/issue.json)
                            echo "$LINE" | grep -qE '^[0-9]+$' || { i=$((i+1)); continue; }

                            jq -n \
                                --arg body "[$(jq -r '.severity' /tmp/issue.json | tr '[:lower:]' '[:upper:]')] [$(jq -r '.category' /tmp/issue.json)] $(jq -r '.title' /tmp/issue.json)

$(jq -r '.detail' /tmp/issue.json)" \
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
                            curl -sf -X POST \\
                                 -H "Authorization: token \$TKN" \\
                                 -H "Content-Type: application/json" \\
                                 -d '{"state":"${state}","context":"claude-ai-review","description":"Multi-agent verdict: ${env.REVIEW_VERDICT}"}' \\
                                 "https://api.github.com/repos/${env.REPO_PATH}/statuses/${env.PR_HEAD_SHA}" || true
                        """
                    }
                }
            }
            cleanWs()
        }
    }
}

// ── Shared helper: runs one specialized agent and saves its raw findings ──────
def runAgent(String agentClass, String mandate) {
    def agentPrompt = """You are a specialized code review agent. Your mandate:

${mandate}

You must ONLY report issues that fall within your mandate — do not report anything outside it.
Be strict about false positives: only flag something if you are confident it is a real issue.

CHANGED FILES: ${env.CHANGED_FILES_LIST}

=== GIT DIFF ===
${readFile('pr_diff.txt')}

${readFile('full_context.txt')}

Return ONLY a JSON array of issues (can be empty), no markdown fences:
[
  {
    "file": "...",
    "line": 1,
    "severity": "critical|high|medium|low",
    "title": "Short title",
    "detail": "Detailed explanation and suggested fix"
  }
]"""

    writeFile file: "prompt_${agentClass}.txt", text: agentPrompt

    withCredentials([string(credentialsId: 'ANTHROPIC_API_KEY', variable: 'CL_KEY')]) {
        sh """
            export ANTHROPIC_API_KEY=\$CL_KEY
            claude -p "\$(cat prompt_${agentClass}.txt)" --output-format json > raw_${agentClass}.json 2>&1 || true
            jq -r '.result // "[]"' raw_${agentClass}.json \
                | sed 's/^```json//; s/^```//; s/```\$//' > agent_${agentClass}.json
            jq empty agent_${agentClass}.json || echo '[]' > agent_${agentClass}.json
        """
    }
}
