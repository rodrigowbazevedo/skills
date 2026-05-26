---
name: plan-to-github-issue
description: Run this skill immediately after every plan is completed and ExitPlanMode is called. Use it to offer saving the plan as a GitHub issue or updating an existing one, and to auto-link the issue into any open PR on the current branch (appends `Closes #<n>` to the PR body). ALWAYS invoke this after ExitPlanMode — do not skip.
---

Run immediately after every ExitPlanMode call, without waiting for the user to ask.

## Steps

### 1. Check GitHub remote
```bash
git remote -v 2>/dev/null | grep -i github
```
If not a git repo with a GitHub remote, stop silently.

### 2. Detect "loaded from issue" case
If the conversation references a GitHub issue number or URL that this plan was loaded from, go to Step 4.

### 3. Offer to create new issue
Ask: "Save this plan as a GitHub issue? (y/n)"

If yes: extract title from the plan's `# Plan: <title>` heading (or ask if missing), then:
```bash
gh issue create --title "Plan: <title>" --body "<full plan content in markdown>"
```
Then create the label if needed and apply it:
```bash
gh label create "plan" --color "#0075ca" --description "Saved planning session" 2>/dev/null || true
gh issue create --title "Plan: <title>" --body "<full plan content in markdown>" --label "plan"
```
Output the created issue URL.

Append the issue number to git config for the current branch so it can be linked when creating a PR. The config holds a comma-separated list (e.g. `42,57`) so multiple plans created on the same branch each contribute a `Closes #N` line later:
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

Then run **Step 5** with `$ISSUE_NUMBER` to back-fill the link into any open PR on this branch.

### 4. Offer to update existing issue
Ask: "Update GitHub issue #<number> with the revised plan? (y/n)"

If yes:
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

Then run **Step 5** with the issue number — Step 5 is idempotent, so it's safe even if the PR already references the issue.

### 5. Auto-link the issue into an open PR (if any)

Runs after Step 3 or Step 4 with `ISSUE_NUMBER` set. If the current branch has an open PR, ensure its body contains `Closes #<ISSUE_NUMBER>`. No prompt — the user has already confirmed the issue create/update; the PR linkage is the natural follow-through.

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

Why this matters: the git-config list (`branch.<name>.github-issue`) is consumed at PR-creation time to seed `Closes` lines. If the PR is already open when the plan/issue is created, that path never runs — so the new issue would be orphaned from the PR. Step 5 closes that gap.

## Notes
- Always use `gh` CLI, never GitHub MCP.
- Do not modify the plan file itself.
- For the `plan` label: suppress errors if it already exists (`2>/dev/null || true`).
- For "loaded from issue" detection: scan conversation for any `#N` reference or `github.com/.*/issues/N` URL.
- Report errors from `gh` and stop.
- Step 5 complements the PR-creation flow in CLAUDE.md (which builds `Closes` lines from `branch.<name>.github-issue` at PR open time): when the PR is already open, Step 5 back-fills the link directly into the existing PR body.
