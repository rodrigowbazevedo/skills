---
name: plan-to-github-issue
description: Saves a completed plan as a GitHub issue (or updates an existing one) and auto-links it into any open PR on the branch (appends `Closes #<n>`). Invoke at two moments. (1) Immediately after every ExitPlanMode call — but DO NOT block asking whether to create the issue; instead record a deferred marker and let implementation start right away. (2) Before the first commit or PR on the branch, when a deferred plan-issue marker is pending — that's when you actually ask the create/update question. Triggers: right after ExitPlanMode; before committing/opening a PR with a pending plan issue; or when the user says "create the issue now" / "save the plan as an issue".
---

This skill runs in **two phases** so plan approval never blocks on a question you won't see for an hour.

- **Phase A — Defer (right after ExitPlanMode):** record that a plan issue is pending, print one line, and hand control back so implementation starts immediately. No blocking question.
- **Phase B — Resolve (before the first commit/PR, or on explicit request):** that's when you're back at the keyboard, so *that's* when you ask whether to create/update the issue, then do it.

Why: a blocking "save as issue? (y/n)" right after plan approval freezes the session — the user walks away expecting work to begin and only sees the prompt much later. Deferring the question to commit/PR time removes the wait entirely while still capturing the issue at a natural checkpoint.

---

## Phase A — Defer (after ExitPlanMode)

### A1. Check GitHub remote
```bash
git remote -v 2>/dev/null | grep -i github
```
If not a git repo with a GitHub remote, stop silently — nothing to defer.

### A2. Determine create vs update
- If the conversation references a GitHub issue number/URL that this plan was **loaded from**, this is the **update** case → marker value `update:<number>`.
- Otherwise it's the **create** case → marker value `create:<plan-file-path>` (the `~/.claude/plans/<file>.md` this plan lives in, so Phase B can read the latest plan content).

Detect "loaded from issue" by scanning the conversation for any `#N` reference or `github.com/.*/issues/N` URL tied to this plan.

### A3. Record the deferred marker (no question, no pause)
Store the intent in branch-scoped git config. It survives context compaction, is branch-scoped, and slash-safe in branch names:
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
git config --local "branch.${BRANCH}.github-issue-deferred" "<create:/path/to/plan.md | update:N>"
```
Then print one line and continue, e.g.:

> Starting implementation — I'll ask about saving this plan as a GitHub issue before we commit or open a PR.

Do **not** call `AskUserQuestion` or any blocking prompt here. Hand control straight back to implementation.

---

## Phase B — Resolve (before first commit/PR, or on explicit request)

Trigger this phase when you are about to make the **first commit or open a PR** on the branch, or when the user explicitly says "create the issue now" / "save the plan as an issue".

### B1. Read the marker
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
DEFERRED=$(git config --local --get "branch.${BRANCH}.github-issue-deferred" 2>/dev/null || true)
```
If empty, nothing is pending — exit silently.

### B2. Ask now (this is the right moment — the user is present and acting)
- **create:** "Save this plan as a GitHub issue? (y/n)"
- **update:** "Update GitHub issue #<number> with the revised plan? (y/n)"

Whatever the answer, **clear the marker** at the end of this phase so it never nags again on this branch:
```bash
git config --local --unset "branch.${BRANCH}.github-issue-deferred" 2>/dev/null || true
```
If the user says **no**, clear the marker and stop — no issue work.

### B3. Create a new issue (create case, on yes)
Extract the title from the plan's `# Plan: <title>` heading (or ask if missing), then create with the `plan` label:
```bash
gh label create "plan" --color "#0075ca" --description "Saved planning session" 2>/dev/null || true
gh issue create --title "Plan: <title>" --body "<full plan content in markdown>" --label "plan"
```
Output the created issue URL.

Append the issue number to git config for the current branch. The config holds a comma-separated list (e.g. `42,57`) so multiple plans on the same branch each contribute a `Closes #N` line later:
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
ISSUE_NUMBER=<number extracted from gh output>
KEY="branch.${BRANCH}.github-issue"
EXISTING=$(git config --local --get "$KEY" 2>/dev/null || true)
if [ -z "$EXISTING" ]; then
  git config --local "$KEY" "$ISSUE_NUMBER"
elif ! echo ",$EXISTING," | grep -q ",$ISSUE_NUMBER,"; then
  git config --local "$KEY" "${EXISTING},${ISSUE_NUMBER}"
fi
```
Then run **B5** with `$ISSUE_NUMBER` to back-fill the link into any open PR on this branch.

### B4. Update an existing issue (update case, on yes)
```bash
gh issue edit <number> --body "<full plan content>"
```
Output the updated issue URL.

Append the issue number to git config for the current branch (comma-separated list, deduped):
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
KEY="branch.${BRANCH}.github-issue"
EXISTING=$(git config --local --get "$KEY" 2>/dev/null || true)
if [ -z "$EXISTING" ]; then
  git config --local "$KEY" "<number>"
elif ! echo ",$EXISTING," | grep -q ",<number>,"; then
  git config --local "$KEY" "${EXISTING},<number>"
fi
```
Then run **B5** with the issue number — B5 is idempotent, so it's safe even if the PR already references the issue.

### B5. Auto-link the issue into an open PR (if any)

Runs after B3 or B4 with `ISSUE_NUMBER` set. If the current branch has an open PR, ensure its body contains `Closes #<ISSUE_NUMBER>`. No prompt — the user has already confirmed the issue create/update; the PR linkage is the natural follow-through.

```bash
# Hardened against the 2026-05-12 near-miss where an empty PR_BODY (caused
# by a chained `jq | jq` choking on literal newlines, with `set -e` off)
# overwrote a real PR body with just "\n\nCloses #N". Three defenses:
#   1. strict mode, so any failure aborts before `gh pr edit` runs;
#   2. body always lives in a file (no shell-var round-trip), extracted
#      once via gh's built-in --jq (no double-jq pipeline);
#   3. a sanity gate that refuses to write if the new body is shorter
#      than the original or no longer contains the original's head.
set -euo pipefail

BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_NUM=$(gh pr list --head "$BRANCH" --state open --json number --jq '.[0].number // empty' 2>/dev/null || true)

if [ -z "${PR_NUM:-}" ]; then
  : # no open PR on this branch — nothing to do
else
  ORIG=$(mktemp -t pr-body-orig.XXXXXX.md)
  gh pr view "$PR_NUM" --json body --jq '.body' > "$ORIG"

  if [ ! -s "$ORIG" ]; then
    # Brand-new PR with empty body: the CLAUDE.md PR-creation flow will
    # seed Closes lines from branch.<name>.github-issue. We won't
    # fabricate a body from nothing here.
    echo "PR #$PR_NUM body is empty; refusing to write a body containing only Closes #$ISSUE_NUMBER (backup: $ORIG)"
  elif grep -qE "(^|[^0-9])Closes #${ISSUE_NUMBER}([^0-9]|$)" "$ORIG"; then
    # Idempotent: non-digit boundaries so `#4` doesn't match `#42`.
    echo "PR #$PR_NUM already references issue #$ISSUE_NUMBER; no edit needed (backup: $ORIG)"
  else
    NEW=$(mktemp -t pr-body-new.XXXXXX.md)
    if grep -qE "^Closes #[0-9]+" "$ORIG"; then
      # Group the new line right after the LAST existing Closes line.
      awk -v line="Closes #${ISSUE_NUMBER}" '
        /^Closes #[0-9]+/ { last=NR }
        { rows[NR]=$0 }
        END {
          for (i=1; i<=NR; i++) {
            print rows[i]
            if (i==last) print line
          }
        }' "$ORIG" > "$NEW"
    else
      { cat "$ORIG"; printf '\n\nCloses #%s\n' "$ISSUE_NUMBER"; } > "$NEW"
    fi

    # Sanity gate — the backstop. New body must be at least as long as
    # the original AND must still contain the original's leading bytes.
    # Either check failing means something corrupted the body; abort
    # loudly with the backup path so recovery is one command away.
    ORIG_BYTES=$(wc -c < "$ORIG")
    NEW_BYTES=$(wc -c  < "$NEW")
    if [ "$NEW_BYTES" -lt "$ORIG_BYTES" ]; then
      echo "ABORT: new PR body ($NEW_BYTES bytes) is shorter than original ($ORIG_BYTES bytes). Backup at $ORIG"
      exit 1
    fi
    HEAD200=$(head -c 200 "$ORIG")
    if ! grep -Fq -- "$HEAD200" "$NEW"; then
      echo "ABORT: new PR body does not contain the first 200 bytes of the original. Backup at $ORIG"
      exit 1
    fi

    # --body-file (not --body): the body never touches argv, so control
    # characters in the body can't break the gh invocation.
    gh pr edit "$PR_NUM" --body-file "$NEW"
    echo "Linked issue #$ISSUE_NUMBER into PR #$PR_NUM (backup: $ORIG)"
  fi
fi
```

Why this matters: the git-config list (`branch.<name>.github-issue`) is consumed at PR-creation time to seed `Closes` lines. If the PR is already open when the issue is created, that path never runs — so the new issue would be orphaned from the PR. B5 closes that gap.

## Notes
- The deferred marker (`branch.<name>.github-issue-deferred`) is the durable handoff between Phase A and Phase B — it must always be cleared in Phase B (B2), on yes or no, so the question never fires twice on the same branch.
- If a commit/PR happens in a fresh session, the marker still tells you a plan issue is pending — read it before committing.
- Always use `gh` CLI, never GitHub MCP.
- Do not modify the plan file itself.
- For the `plan` label: suppress errors if it already exists (`2>/dev/null || true`).
- For "loaded from issue" detection: scan conversation for any `#N` reference or `github.com/.*/issues/N` URL.
- Report errors from `gh` and stop.
- B5 complements the PR-creation flow in CLAUDE.md (which builds `Closes` lines from `branch.<name>.github-issue` at PR open time): when the PR is already open, B5 back-fills the link directly into the existing PR body.
