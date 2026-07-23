---
name: plan-to-github-issue
description: Saves a completed plan as a GitHub issue (create a new one, attach a sub-issue, or add a comment) and auto-links it into any open PR on the branch (appends `Closes #<n>`). Invoke at two moments. (1) Immediately after every ExitPlanMode call — but DO NOT block asking whether to create the issue; instead record a deferred marker and let implementation start right away. (2) Before the first commit or PR on the branch, when a deferred plan-issue marker is pending — that's when you ask how to save the plan. Triggers: right after ExitPlanMode; before committing/opening a PR with a pending plan issue; or when the user says "create the issue now" / "save the plan as an issue".
---

This skill runs in **two phases** so plan approval never blocks on a question you won't see for an hour.

- **Phase A — Defer (right after ExitPlanMode):** record that a plan issue is pending, print one line, and hand control back so implementation starts immediately. No blocking question.
- **Phase B — Resolve (before the first commit/PR, or on explicit request):** that's when you're back at the keyboard, so *that's* when you ask **how** to save the plan, then do it.

Why: a blocking "save as issue?" right after plan approval freezes the session — the user walks away expecting work to begin and only sees the prompt much later. Deferring the question to commit/PR time removes the wait entirely while still capturing the issue at a natural checkpoint.

A plan isn't always a brand-new issue. It might hang off an existing issue you started work from, or be the second plan on a branch that already has one. Phase A only records *context* (which issue, if any, the plan came from); Phase B is where you and the user decide the disposition, because that's a judgement call best made with the user present.

**Never edit an existing issue's body.** When a plan started from / was loaded against an existing issue, that issue's body is the user's — clobbering it (overwrite, "replace body", "append section to body") is exactly what this skill must not do. A plan attaches to an existing issue **only** as a sub-issue or a comment. The only body this skill ever writes is one on a brand-new issue it creates itself.

---

## Phase A — Defer (after ExitPlanMode)

### A1. Check GitHub remote
```bash
git remote -v 2>/dev/null | grep -i github
```
If not a git repo with a GitHub remote, stop silently — nothing to defer.

### A2. Record the deferred marker (no question, no pause)
Phase A does **not** decide create-vs-update-vs-anything. It records only where the plan came from, so Phase B can classify with the user present.

- Always start the marker with `create:<plan-file-path>` — the `~/.claude/plans/<file>.md` this plan lives in, so Phase B can read the latest plan content.
- If this plan was **loaded from / started against an existing GitHub issue**, append `|src:<number>`. Detect this by scanning the conversation for any `#N` reference or `github.com/.*/issues/N` URL tied to this plan.

Store it in branch-scoped git config — it survives context compaction and is branch-scoped:
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
git config --local "branch.${BRANCH}.github-issue-deferred" "create:/path/to/plan.md"        # no source issue
# or, when the plan came from an existing issue:
git config --local "branch.${BRANCH}.github-issue-deferred" "create:/path/to/plan.md|src:123"
```
Then print one line and continue, e.g.:

> Starting implementation — I'll ask how to save this plan as a GitHub issue before we commit or open a PR.

Do **not** call `AskUserQuestion` or any blocking prompt here. Hand control straight back to implementation.

---

## Phase B — Resolve (before first commit/PR, or on explicit request)

Trigger this phase when you're about to make the **first commit or open a PR** on the branch, or when the user explicitly says "create the issue now" / "save the plan as an issue".

### B1. Read the marker and prior issues
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
DEFERRED=$(git config --local --get "branch.${BRANCH}.github-issue-deferred" 2>/dev/null || true)
PRIOR=$(git config --local --get "branch.${BRANCH}.github-issue" 2>/dev/null || true)   # comma list of plan issues already on this branch
```
If `DEFERRED` is empty, nothing is pending — exit silently.

Parse `DEFERRED`:
- `create:<plan-path>` → plan path, no source issue.
- `create:<plan-path>|src:<N>` → plan path + source issue `N`.
- Legacy `update:<N>` (older marker format) → treat as source issue `N` with no reliable plan path; read the plan from the current conversation instead.

### B2. Classify the context
Decide which situation you're in — this selects the option set in B3. No need to inspect the issue's labels: a plan never edits an existing body, so it doesn't matter whether the source issue is itself a saved plan or a feature/bug.

1. **Source issue `N` present** (the plan started from / was loaded against issue `N`) → **started from an existing issue**.
2. **No source issue, but `PRIOR` is non-empty** → **subsequent plan** on a branch that already has a plan issue (call the most recent one `M`). This is a fresh plan produced mid-implementation.
3. **Neither** → **fresh plan** (the common create case).

### B3. Ask how to save it (use `AskUserQuestion`)
Present the option set for the context. **Auto-preselect** the recommended option (listed first) — for the subsequent-plan case, read the new plan against issue `M` and preselect *new standalone issue* if it's unrelated work or *sub-issue under #M* if it's a complement; the user can always override.

| Context | Options (recommended first) |
|---|---|
| **Fresh plan** | Create new issue · Discard |
| **Started from an existing issue #N** | Sub-issue under #N · Comment on #N · Discard (#N already tracks it) |
| **Subsequent plan (prior #M)** | *auto-preselected:* New standalone issue *or* Sub-issue under #M · Comment on #M · Discard |

Do **not** offer "update body", "replace body", "append section to body", or any variant that writes to an existing issue's body — see the rule at the top of this file. The only body-writing option is *Create new issue*, and only in the fresh-plan context.

### B4. Execute the chosen disposition
Read `references/dispositions.md` and follow the matching section. In brief:

- **Create new issue** — the only body this skill writes; record the issue number for PR linking.
- **Sub-issue under #P** — create a child issue (always with the `plan` label, even under a non-plan parent) and attach it to `#P`; record the **child** number for PR linking.
- **Comment on #C** — add the plan as a new comment (never touches #C's body); then **ask** whether the PR should also `Closes #C`.
- **Discard** — create nothing, but if a source/parent issue exists, still record it so the PR closes it (the initial issue tracks the work).

Every disposition finishes by clearing the deferred marker so the question never fires twice on this branch:
```bash
git config --local --unset "branch.${BRANCH}.github-issue-deferred" 2>/dev/null || true
```
Clear it even when the user picks Discard or answers "no".

---

## Notes
- The deferred marker (`branch.<name>.github-issue-deferred`) is the durable handoff between Phase A and Phase B — always clear it in Phase B, so the question never fires twice on the same branch.
- If a commit/PR happens in a fresh session, the marker still tells you a plan issue is pending — read it before committing.
- The `branch.<name>.github-issue` list (comma-separated, deduped) is what the CLAUDE.md PR-creation flow reads to seed `Closes #N` lines. Sub-issues record the **child** number; comments don't touch it (unless you opt in per B4); discard records the source issue.
- Always use `gh` CLI (`gh issue`, `gh api`), never GitHub MCP.
- Do not modify the plan file itself.
- Report errors from `gh` and stop.

## Reference Index

| File | Purpose | When to read |
|------|---------|--------------|
| `references/dispositions.md` | Bash for each Phase B disposition (create, sub-issue, comment, discard) + the hardened PR-linking (`Closes #N`) routine | In Phase B once the user has chosen a disposition. Phase-A-only invocations never need it. |
