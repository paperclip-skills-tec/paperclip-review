---
name: paperclip-review
description: >
  Dispatch parallel multi-agent code review on a PR issue. Evaluates trigger
  conditions (priority, file count, security relevance) and dispatches up to 5
  specialised review sub-agents — Correctness, Quality, Architecture, Test
  Coverage, and Security — then aggregates their findings into a single summary
  comment. Use when an issue enters review and a structured code review is needed.
---

# Paperclip Review — Multi-Agent Code Review Dispatch

This skill dispatches parallel code review sub-agents on a PR and aggregates results.

## When to Invoke

Call this skill when a task enters `in_review` and you need to determine the appropriate review depth. The skill evaluates trigger conditions and dispatches the correct review tier.

## Prerequisites

- `PAPERCLIP_API_KEY`, `PAPERCLIP_API_URL`, `PAPERCLIP_COMPANY_ID`, `PAPERCLIP_AGENT_ID`, `PAPERCLIP_RUN_ID` must be set.
- The calling agent must have the issue checked out.
- The issue should reference a code change (branch, PR, or diff).

## Configuration

Review agent IDs are resolved from the company agent pool by matching agents whose name contains one of the 5 review dimension keywords. The skill reads agent configuration at runtime:

```bash
REVIEW_AGENTS_JSON=$(curl -sS "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/agents" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" | python3 -c '
import json, sys
agents = json.load(sys.stdin)
dims = {"correctness": None, "quality": None, "architecture": None, "test coverage": None, "security": None}
for a in agents:
    name_lower = a.get("name","").lower()
    for dim in dims:
        if dim in name_lower and dims[dim] is None:
            dims[dim] = {"id": a["id"], "name": a["name"]}
            break
print(json.dumps(dims))
')
```

If any dimension agent is missing, fall back to assigning all review tasks to the Dev Lead (`5a301e20-07a8-4a34-b113-4818437c02ac`).

## Trigger Evaluation

Evaluate these conditions **in order**. The first match wins:

### Step 1 — Gather inputs

```bash
ISSUE_ID="<the-issue-being-reviewed>"

ISSUE_JSON=$(curl -sS "$PAPERCLIP_API_URL/api/issues/$ISSUE_ID" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY")

PRIORITY=$(echo "$ISSUE_JSON" | python3 -c 'import json,sys; print(json.load(sys.stdin).get("priority","medium"))')
```

Determine the changed files. If the issue has an execution workspace with a git branch, diff against the base branch:

```bash
CHANGED_FILES=$(git diff --name-only origin/main...HEAD 2>/dev/null || echo "")
FILE_COUNT=$(echo "$CHANGED_FILES" | grep -c . 2>/dev/null || echo "0")
```

### Step 2 — Classify security-relevant files

A file is security-relevant if its path matches any of these patterns:

```
**/auth/**
**/authorization/**
**/authz/**
**/permissions/**
**/rbac/**
**/acl/**
**/payments/**
**/billing/**
**/stripe/**
**/secrets/**
**/crypto/**
**/encryption/**
**/middleware/auth*
**/middleware/session*
**/data-access/**
**/dal/**
**/.env*
**/credentials*
**/tokens/**
```

```bash
SECURITY_PATTERNS="auth/|authorization/|authz/|permissions/|rbac/|acl/|payments/|billing/|stripe/|secrets/|crypto/|encryption/|middleware/auth|middleware/session|data-access/|dal/|\.env|credentials|tokens/"
HAS_SECURITY_FILES=$(echo "$CHANGED_FILES" | grep -iE "$SECURITY_PATTERNS" | head -1)
```

### Step 3 — Determine review tier

| Condition | Tier |
|-----------|------|
| `PRIORITY` is `critical` or `high` | **full** (5-agent) |
| `FILE_COUNT` >= 3 | **full** (5-agent) |
| `HAS_SECURITY_FILES` is non-empty | **full** (5-agent) |
| `PRIORITY` is `medium` AND `FILE_COUNT` < 3 AND no security files | **single** (1-agent) |
| `PRIORITY` is `low` | **checklist** (no agents) |

```bash
if [ "$PRIORITY" = "critical" ] || [ "$PRIORITY" = "high" ]; then
  REVIEW_TIER="full"
elif [ "$FILE_COUNT" -ge 3 ]; then
  REVIEW_TIER="full"
elif [ -n "$HAS_SECURITY_FILES" ]; then
  REVIEW_TIER="full"
elif [ "$PRIORITY" = "medium" ]; then
  REVIEW_TIER="single"
else
  REVIEW_TIER="checklist"
fi
```

## Dispatch — Full Review (5 agents)

Create 5 child issues in parallel, one per review dimension. Each child issue:
- Has `parentId` set to the review coordination issue
- Has `inheritExecutionWorkspaceFromIssueId` set to the original PR issue so agents see the code
- Is assigned to the corresponding review agent

### Review Dimensions

| # | Dimension | Focus |
|---|-----------|-------|
| 1 | **Correctness** | Does the implementation match the spec/acceptance criteria? Logic errors, edge cases, incorrect assumptions? |
| 2 | **Quality** | Code quality: naming, structure, SOLID principles, duplication, readability, maintainability |
| 3 | **Architecture** | Layer violations, unexpected dependencies, pattern deviations, structural anti-patterns |
| 4 | **Test Coverage** | Are new/changed behaviours covered? Are tests meaningful? Missing edge cases? |
| 5 | **Security** | OWASP top 10, injection risks, auth/authz issues, secrets handling, input validation |

### Dispatch Script

```bash
COMPANY_ID="$PAPERCLIP_COMPANY_ID"
PR_ISSUE_ID="<the-original-PR-issue>"
PARENT_ISSUE_ID="$ISSUE_ID"
GOAL_ID="<goal-id-from-parent>"
BRANCH_NAME="<branch-being-reviewed>"

DIMENSIONS=("correctness" "quality" "architecture" "test-coverage" "security")
FOCUS_DESCRIPTIONS=(
  "Review for correctness: Does the implementation match the spec acceptance criteria? Are there logic errors, edge cases, or incorrect assumptions? Check each acceptance criterion against the actual code changes."
  "Review for code quality: Evaluate naming conventions, code structure, SOLID principles, duplication, readability, and maintainability. Flag any anti-patterns or style violations against project conventions."
  "Review for architecture: Check for layer violations, unexpected dependencies, pattern deviations from the existing codebase, and structural anti-patterns. Verify the change fits the existing module boundaries."
  "Review for test coverage: Are new or changed behaviours covered by tests? Are the tests meaningful and not just smoke tests? Identify missing edge cases or integration test gaps."
  "Review for security: OWASP top 10 awareness — check for injection risks (SQL, XSS, command), auth/authz issues, secrets handling, input validation, and unsafe data flows."
)

for i in "${!DIMENSIONS[@]}"; do
  DIM="${DIMENSIONS[$i]}"
  FOCUS="${FOCUS_DESCRIPTIONS[$i]}"

  # Resolve agent ID for this dimension from REVIEW_AGENTS_JSON
  AGENT_ID=$(echo "$REVIEW_AGENTS_JSON" | python3 -c "
import json, sys
data = json.loads(sys.stdin.read())
dim_key = '$DIM'.replace('-', ' ')
entry = data.get(dim_key, {})
print(entry.get('id', '') if entry else '')
")

  # Fall back to Dev Lead if no specialised agent
  if [ -z "$AGENT_ID" ]; then
    AGENT_ID="5a301e20-07a8-4a34-b113-4818437c02ac"
  fi

  curl -sS -X POST "$PAPERCLIP_API_URL/api/companies/$COMPANY_ID/issues" \
    -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
    -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
    -H "Content-Type: application/json" \
    -d "$(cat <<PAYLOAD
{
  "title": "Code Review [$DIM]: $BRANCH_NAME",
  "description": "## Review Task\n\n$FOCUS\n\n### Changed Files\n\n$(echo "$CHANGED_FILES" | sed 's/^/- /')\n\n### Instructions\n\n1. Check out the branch and read the diff\n2. Review against your dimension focus above\n3. Post your findings as a structured comment on this issue\n4. Mark this issue as \`done\` when complete\n\n### Output Format\n\nPost a comment with:\n- **Severity**: critical / warning / suggestion / note\n- **File:Line**: location of the finding\n- **Finding**: description of the issue\n- **Recommendation**: suggested fix or improvement",
  "parentId": "$PARENT_ISSUE_ID",
  "goalId": "$GOAL_ID",
  "assigneeAgentId": "$AGENT_ID",
  "inheritExecutionWorkspaceFromIssueId": "$PR_ISSUE_ID",
  "priority": "high",
  "status": "todo"
}
PAYLOAD
)" &
done
wait
```

All 5 issues are created in parallel (background `&` + `wait`). Paperclip wakes each assigned agent to start their review.

## Dispatch — Single Review (1 agent)

For `medium` priority, single-file PRs with no security-relevant changes, dispatch only the **Quality** agent (or Dev Lead fallback) with a combined review prompt covering all 5 dimensions at a lighter depth:

```bash
AGENT_ID=$(echo "$REVIEW_AGENTS_JSON" | python3 -c "
import json, sys
data = json.loads(sys.stdin.read())
entry = data.get('quality', {})
print(entry.get('id', '') if entry else '')
") 

if [ -z "$AGENT_ID" ]; then
  AGENT_ID="5a301e20-07a8-4a34-b113-4818437c02ac"
fi

curl -sS -X POST "$PAPERCLIP_API_URL/api/companies/$COMPANY_ID/issues" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  -d "$(cat <<PAYLOAD
{
  "title": "Code Review [combined]: $BRANCH_NAME",
  "description": "## Single-Agent Review\n\nThis is a lighter review for a medium-priority, small-scope change.\n\nReview across all dimensions at surface level:\n1. **Correctness** — does it match the spec?\n2. **Quality** — naming, structure, readability\n3. **Architecture** — does it fit existing patterns?\n4. **Test Coverage** — are changes tested?\n5. **Security** — any obvious risks?\n\n### Changed Files\n\n$(echo "$CHANGED_FILES" | sed 's/^/- /')\n\n### Output Format\n\nPost a comment with findings using severity/file/finding/recommendation format.",
  "parentId": "$PARENT_ISSUE_ID",
  "goalId": "$GOAL_ID",
  "assigneeAgentId": "$AGENT_ID",
  "inheritExecutionWorkspaceFromIssueId": "$PR_ISSUE_ID",
  "priority": "medium",
  "status": "todo"
}
PAYLOAD
)"
```

## Dispatch — Checklist Only (no agents)

For `low` priority tasks, post a self-reflection checklist as a comment on the issue instead of dispatching agents:

```bash
CHECKLIST_COMMENT=$(cat <<'MD'
## Self-Review Checklist

Before merging this low-priority change, verify:

- [ ] Code compiles and tests pass
- [ ] No unintended side effects in adjacent code
- [ ] Variable and function names are clear
- [ ] No hardcoded secrets or credentials
- [ ] No console.log / debug statements left in
- [ ] Changes match the issue description
- [ ] If touching shared code, no breaking changes for other consumers

No agent review dispatched — this is a low-priority reflection checklist.
MD
)

curl -sS -X POST "$PAPERCLIP_API_URL/api/issues/$ISSUE_ID/comments" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg body "$CHECKLIST_COMMENT" '{body: $body}')"
```

## Aggregation — Collecting Review Results

After dispatching the full 5-agent review, the dispatching agent should **not** poll for completion. Instead, rely on Paperclip's `issue_children_completed` wake event on the parent issue.

When woken by `issue_children_completed`:

### Step 1 — Fetch child issue comments

```bash
CHILDREN=$(curl -sS "$PAPERCLIP_API_URL/api/companies/$COMPANY_ID/issues?parentId=$PARENT_ISSUE_ID" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY")

# For each child, get its comments (the review findings)
echo "$CHILDREN" | python3 -c '
import json, sys, subprocess

children = json.load(sys.stdin)
findings = {}
for child in children:
    cid = child["id"]
    title = child["title"]
    dim = title.split("[")[1].split("]")[0] if "[" in title else "unknown"
    
    result = subprocess.run(
        ["curl", "-sS",
         f"'"$PAPERCLIP_API_URL"'/api/issues/{cid}/comments",
         "-H", "Authorization: Bearer '"$PAPERCLIP_API_KEY"'"],
        capture_output=True, text=True
    )
    comments = json.loads(result.stdout)
    # Take the last substantive comment as the review output
    review_comments = [c for c in comments if len(c.get("body","")) > 50]
    if review_comments:
        findings[dim] = review_comments[-1]["body"]

print(json.dumps(findings, indent=2))
' > /tmp/review-findings.json
```

### Step 2 — Aggregate into summary comment

```bash
SUMMARY=$(python3 -c '
import json

with open("/tmp/review-findings.json") as f:
    findings = json.load(f)

lines = ["## Code Review Summary\n"]
lines.append(f"**Dimensions reviewed:** {len(findings)}/5\n")

critical_count = 0
warning_count = 0
suggestion_count = 0

for dim, body in sorted(findings.items()):
    lines.append(f"### {dim.title()}\n")
    lines.append(body)
    lines.append("")
    
    body_lower = body.lower()
    critical_count += body_lower.count("critical")
    warning_count += body_lower.count("warning")
    suggestion_count += body_lower.count("suggestion")

lines.insert(2, f"**Findings:** {critical_count} critical, {warning_count} warnings, {suggestion_count} suggestions\n")

if critical_count > 0:
    lines.append("---\n")
    lines.append("**Action required:** Critical findings must be addressed before merge.\n")
elif warning_count > 0:
    lines.append("---\n")
    lines.append("**Recommendation:** Address warnings before merge where feasible.\n")
else:
    lines.append("---\n")
    lines.append("**Result:** No critical or warning findings. Approved for merge.\n")

print("\n".join(lines))
')

# Post aggregated summary on the original PR issue
curl -sS -X POST "$PAPERCLIP_API_URL/api/issues/$PR_ISSUE_ID/comments" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg body "$SUMMARY" '{body: $body}')"
```

## Complete Workflow Summary

```
1. Agent invokes paperclip-review skill on a PR issue
2. Skill gathers: priority, changed files, security file check
3. Skill determines tier: full | single | checklist
4. Dispatch:
   - full  → 5 child issues (one per dimension), assigned to review agents
   - single → 1 child issue (combined review), assigned to quality agent
   - checklist → comment posted on issue, no agents dispatched
5. Wait for issue_children_completed wake (full/single only)
6. Aggregate findings from child issue comments
7. Post summary comment on the original PR issue
```

## Error Handling

- If no review agents are found, all review tasks fall back to the Dev Lead.
- If a child review agent fails (issue stays `in_progress` past SLA), the dispatching agent should escalate to the Dev Lead.
- If `git diff` fails (no execution workspace), post a comment asking the PR author to provide the branch name and changed file list manually.
