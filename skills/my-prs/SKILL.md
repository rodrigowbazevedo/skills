---
name: my-prs
description: >
  Lists the current user's open PRs across active Claravine (TrackingFirst) repos
  with per-PR review status, grouped by whether they're approved, have changes
  requested, or are still waiting on reviewers. Use when the user says
  "list my PRs", "my PR queue", "my pull requests", "my review queue", "what
  PRs do I have open", "show PRs ready for review", "my approved PRs",
  "claravine PRs", "PR status", "PRs needing review", "PRs waiting on me",
  or any variant of asking about their own open PRs at Claravine / TrackingFirst.
---

# My PRs

Lists open PRs in the `TrackingFirst` GitHub org that the current user authored or is assigned to, annotated with review decision so it's easy to see what's ready to merge vs still waiting on reviewers.

## Step 1 — Resolve the current user

```bash
gh api user --jq '.login'
```

Cache the returned login as `USER_LOGIN` for the next step.

## Step 2 — Fetch PRs with review status

Use the GraphQL search API — it returns `reviewDecision`, `repository.isArchived`, `mergeStateStatus`, and `mergeable` inline, which `gh search prs` does not. Run the two queries **in parallel** (two Bash tool calls in the same message) to keep the skill fast.

**Authored query:**

```bash
gh api graphql -f query='
  query($q: String!) {
    search(query: $q, type: ISSUE, first: 100) {
      nodes {
        ... on PullRequest {
          number title url isDraft createdAt updatedAt
          headRefName
          reviewDecision
          mergeStateStatus mergeable
          autoMergeRequest { mergeMethod }
          author { login }
          repository { name nameWithOwner isArchived }
        }
      }
    }
  }
' -f q="is:pr is:open org:TrackingFirst author:<USER_LOGIN>"
```

`headRefName` is the PR's source branch (e.g. `feat/jsx-to-tsx-batch-9`). Used in expanded rendering.

`mergeStateStatus` and `mergeable` are computed lazily by GitHub. A PR with no recent activity may return `UNKNOWN` on the first query while computation runs server-side. Treat `UNKNOWN` as "no indicator" rather than retrying — keeping the skill to a single round-trip is more important than coverage on the long tail.

**Assigned query:** same shape, but with `assignee:<USER_LOGIN>` instead of `author:`.

**Opt-in third query** — only run if the user's prompt asks for review-requested PRs (phrases like "include my review queue", "PRs waiting on me", "add the ones I need to review"):
`review-requested:<USER_LOGIN>`. Skip this query by default.

## Step 3 — Merge, dedupe, filter

- **Dedupe** by `url` — a PR can appear in both the authored and assigned result sets.
- **Exclude drafts** (`isDraft: true`) unless the user said "include drafts".
- **Exclude archived repos** (`repository.isArchived: true`). Always. GraphQL tells you directly — no need to consult a separate archived list.
- **Tag each PR** with its source relationship to the user: `authored`, `assigned` (assignee but not author), or `review-requested` (only when Step 2's opt-in query ran).

## Step 4 — Group by review decision

Render three buckets, in this order:

1. **✅ Approved** — `reviewDecision == "APPROVED"` → ready to merge, action is on the user
2. **🔴 Changes Requested** — `reviewDecision == "CHANGES_REQUESTED"` → action is on the user
3. **⏳ Review Required** — `reviewDecision == "REVIEW_REQUIRED"` or `null` → waiting on reviewers

Within each bucket, sort by `updatedAt` newest-first. If any PRs are `assigned` (not authored), list them under a sub-heading "Assigned to you" within each bucket so the user can see which ones are someone else's work they took over. If the opt-in review-requested query ran, render those as a **fourth** group: **🔵 Awaiting your review** (separate from the three decision buckets, since these are other people's PRs waiting on the user).

## Step 5 — Render

Two rendering modes. **Expanded is the default** — it has room for the branch name, which is often long. **Compact** matches the original single-line format and is opt-in via `--compact` or natural-language synonyms (see Step 6).

In both modes the PR identifier is a markdown link (`[<repo>#<number>](<url>)`) so it stays clickable in rendered output.

### Expanded (default)

Each PR is its own **paragraph** (not a bullet-list item), with a **blank line between PRs**. Use `<br>` to force the line breaks inside each paragraph — without it, the three lines collapse into a single soft-wrapped paragraph; without paragraphs (i.e. if you use a `-` bullet list instead), terminal renderers fold the inter-item blank lines and the blocks run together.

```
**[<repo>#<number>](<url>)** — *<Mon DD>* [auto-merge] [💥 conflicts] [🔄 needs update] [⚠️ stale]<br>
<title><br>
`<headRefName>`

**[<repo>#<number>](<url>)** — *<Mon DD>* [tags...]<br>
<title><br>
`<headRefName>`
```

The header line (PR link + date + tags) carries the at-a-glance state. The title gets its own line so it isn't truncated. The branch sits alone in backticks so it's easy to copy for `git checkout` and stays readable no matter how long it is.

### Compact (`--compact`)

One line per PR, no branch — identical to the pre-expanded format so users who prefer dense output get exactly what they had:

```
- [<repo>#<number>](<url>) — <title> — *<Mon DD>* [auto-merge] [💥 conflicts] [🔄 needs update] [⚠️ stale]
- [<repo>#<number>](<url>) — <title> — *<Mon DD>* [tags...]
```

No blank lines between PRs in compact mode — single-line entries already read as a list.

### Tags (both modes)

Each tag is appended only when its condition holds:

- **`[auto-merge]`** — `autoMergeRequest` is non-null. GitHub will merge the PR automatically once required checks/reviews pass. An approved + auto-merge PR needs no action — it'll merge itself.
- **`[💥 conflicts]`** — `mergeable == "CONFLICTING"`. The PR has merge conflicts that must be resolved manually before it can merge. This is the more urgent of the two branch-state signals, so render it instead of `[🔄 needs update]` when both could apply.
- **`[🔄 needs update]`** — `mergeStateStatus == "BEHIND"` AND `mergeable != "CONFLICTING"`. The base branch has commits the PR branch doesn't — equivalent to GitHub's "Update branch" button being shown. Skip this tag when conflicts are already flagged, since you can't update a conflicted branch without resolving the conflicts first.
- **`[⚠️ stale]`** — `updatedAt` is more than 30 days ago.

If `mergeStateStatus` or `mergeable` come back as `UNKNOWN`, render no branch-state tag for that PR.

End with a one-line summary:

```
**Summary:** N approved, M changes requested, K awaiting review.
```

## Step 6 — Honor natural-language filters

Interpret the user's prompt — mostly natural language, with one literal token (`--compact`) recognized for the rendering mode. Examples:

- *"my approved PRs"* / *"ready to merge"* → render only the **Approved** bucket
- *"PRs needing review"* / *"waiting on reviewers"* → render only the **Review Required** bucket
- *"include drafts"* → don't filter drafts in Step 3
- *"include my review queue"* / *"PRs waiting on me"* → also run the opt-in `review-requested:` query in Step 2 and render the 4th group
- *"--compact"* / *"compact"* / *"one-line"* / *"single line"* / *"tldr"* / *"dense"* / *"short"* → render in **Compact** mode (Step 5). Default is Expanded.
- No filter phrase → render all three decision buckets in full, Expanded mode.

Filters compose: *"my approved PRs --compact"* renders only the Approved bucket in compact form.

If the user's phrasing is ambiguous between two filters, render the full list and note in one line which filters you considered.

## Why GraphQL and not `gh search prs`

`gh search prs` doesn't expose `reviewDecision` or `repository.isArchived`. Falling back to it would require one `gh pr view --json reviewDecision` call per PR to recover the status — for 15 PRs that's 15 extra round trips. The single GraphQL query returns everything needed and also filters archived repos without maintaining a separate list.
