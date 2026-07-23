# Phase B dispositions

Bash for each disposition chosen in Phase B, plus shared helpers. All examples assume:
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
```
`gh api` auto-substitutes `{owner}` and `{repo}` from the current repo — leave those literal in the paths below.

Extract the plan **title** from the plan's `# Plan: <title>` heading (or ask if missing). The plan **body** is the full markdown of the plan file (`create:<plan-path>` from the marker); for a legacy `update:N` marker with no path, use the plan content from the conversation.

---

## Shared helper — record an issue number for PR linking

Appends `NUMBER` to the branch's comma-separated, deduped `github-issue` list. The CLAUDE.md PR-creation flow reads this to seed `Closes #N` lines.
```bash
record_issue() {
  local number="$1"
  local key="branch.${BRANCH}.github-issue"
  local existing
  existing=$(git config --local --get "$key" 2>/dev/null || true)
  if [ -z "$existing" ]; then
    git config --local "$key" "$number"
  elif ! printf ',%s,' "$existing" | grep -q ",$number,"; then
    git config --local "$key" "${existing},${number}"
  fi
}
```

---

## Create new issue

```bash
gh label create "plan" --color "0075ca" --description "Saved planning session" 2>/dev/null || true
URL=$(gh issue create --title "Plan: <title>" --body-file "<plan-path>" --label "plan")
echo "$URL"
NUMBER=$(basename "$URL")
record_issue "$NUMBER"
```
Then run **PR linking** with `$NUMBER`.

---

## Update #N body

`N` is a plan issue you loaded from; overwrite its body with the revised plan.
```bash
gh issue edit N --body-file "<plan-path>"
gh issue view N --json url --jq '.url'
record_issue "N"
```
Then run **PR linking** with `N`.

> Only ever overwrite a body when the target carries the `plan` label. Never `gh issue edit --body` a real feature/bug issue — use a sub-issue or comment instead.

---

## Sub-issue under #P

Create a child plan issue and attach it to parent `#P`. The child **always** gets the `plan` label, even when `#P` is a non-plan feature/bug issue, so plan-generated issues stay filterable.

```bash
gh label create "plan" --color "0075ca" --description "Saved planning session" 2>/dev/null || true
CHILD_URL=$(gh issue create --title "Plan: <title>" --body-file "<plan-path>" --label "plan")
CHILD_NUM=$(basename "$CHILD_URL")

# The sub-issues REST endpoint needs the child's internal integer id (NOT its number).
CHILD_ID=$(gh api "repos/{owner}/{repo}/issues/${CHILD_NUM}" --jq '.id')

gh api --method POST "repos/{owner}/{repo}/issues/P/sub_issues" -F "sub_issue_id=${CHILD_ID}"

echo "Created sub-issue $CHILD_URL under #P"
record_issue "$CHILD_NUM"
```
Then run **PR linking** with `$CHILD_NUM` (the PR closes the sub-issue — the concrete unit of work — not the parent).

---

## Comment on #C

Append the plan as a comment; the plan is a complement to `#C`, not a separately-closeable unit.
```bash
gh issue comment C --body-file "<plan-path>"
```
Then **ask** the user (via `AskUserQuestion` or a short prompt): *"Should the PR also close #C?"*
- **Yes** → `record_issue "C"` and run **PR linking** with `C`.
- **No** (default) → leave the `github-issue` list untouched. Done.

---

## Discard

Create nothing. But if a source/parent issue is in play (the `src:<N>` from the marker, or the parent in a revise/subsequent context), still link it so the PR closes the initial issue that tracks the work:
```bash
# only if a source/parent issue number is known:
record_issue "<source-issue-number>"
```
Then run **PR linking** with that number (skip entirely if there's no source issue — a plain fresh-plan discard links nothing).

---

## PR linking — ensure the open PR contains `Closes #<ISSUE_NUMBER>`

Runs after a disposition with `ISSUE_NUMBER` set. If the current branch has an open PR, ensure its body contains `Closes #<ISSUE_NUMBER>`. No prompt — the user has already confirmed the disposition; the PR linkage is the natural follow-through. Idempotent and safe to run repeatedly.

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

Why this matters: the git-config list (`branch.<name>.github-issue`) is consumed at PR-creation time to seed `Closes` lines. If the PR is already open when the issue is created, that path never runs — so the new issue would be orphaned from the PR. This routine closes that gap.
