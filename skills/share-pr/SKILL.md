---
name: share-pr
description: Shares a GitHub PR to Slack channels, DMs specific people, and requests GitHub PR reviews. Use when the user says "share PR", "send PR to Slack", "post PR", "share pull request", "notify team about PR", "send PR to <person>", "request review from", "notify <person>", "DM <person> about PR", or any variation of sharing a PR on Slack or with team members.
metadata:
  mcp-server: [slack, atlassian]
---

# Share PR

## Prerequisites

- **Slack MCP** (required) — used by step 7 to deliver channel posts and DMs (`slack_send_message`). Without it, the skill can compose and preview the message but cannot send it.
- **Atlassian MCP** (optional) — used by step 3 to enrich the message with Jira ticket context (`getJiraIssue`). Without it, the Jira block is silently skipped.
- **`gh` CLI** — used for all GitHub operations (`gh pr view`, `gh pr diff`, `gh pr edit --add-reviewer`). Not an MCP — install separately.
- **`slack-channels` and `team-ref` skills** — invoked from step 5 to resolve channel and person targets. If they're not installed, ask the user to install them first or provide channel IDs / Slack user IDs inline.

## Step 1 — Detect current PR

- If the user provided a PR number or URL, run: `gh pr view <ref> --json url,title,body,number,headRefName`
- Otherwise run: `gh pr view --json url,title,body,number,headRefName`
- If no PR is found, inform the user and stop.
- Save the PR title, URL, body, and number for later steps.

## Step 2 — Gather PR changes

- Run `gh pr diff` (or `gh pr diff <number>`) to get the code diff.
- Read the diff to understand what files changed and the nature of the changes.
- Combine with the PR body for full context.

## Step 3 — Look up Jira ticket (optional)

If a `jira` skill is installed in this environment, invoke it to extract the ticket ID from the PR's `headRefName` (with fallback to PR title/body), fetch ticket details, and resolve the ticket URL. The `jira` skill handles MCP setup, ticket-key regex, and URL resolution.

From the ticket data, note:

- **Is Bug**: `fields.issuetype.name` equals `"Bug"`
- **Is Release Blocker**: `fields.labels` contains `"release_blocker"` (only meaningful if your team uses that label)
- **Ticket URL**: from the ticket response

If no `jira` skill is available, no ticket is found, or the fetch fails, continue without Jira context — do not block the sharing flow.

## Step 4 — Compose the Slack message

**Slack mrkdwn ≠ GitHub markdown.** In Slack: `*text*` = **bold**, `_text_` = *italic*, `**text**` renders as literal asterisks. Section labels and the PR title in this skill are always **bold** (`*...*`), never italic (`_..._`). Mixing these up has shipped a malformed DM before.

Synthesize from both the PR body AND the actual diff. Format the message using this structure — **every line that contains a URL must be followed by a blank line**:

```
{bug-prefix}*{PR title}* — {PR URL}

{release-blocker-line}

{ticket-line}

*What changed:* {brief summary of code changes derived from the diff}

*Why:* {motivation/reason derived from PR body + diff context}
```

When `{release-blocker-line}` or `{ticket-line}` don't apply, omit them and their surrounding blank lines — that optional block collapses to nothing. However, these three blank lines are **always required** regardless of whether the optional lines are present:

1. Blank line after the PR URL line (also required by the link-breaking rule)
2. Blank line before `*What changed:*`
3. Blank line before `*Why:*`

These spacing rules exist for readability — without them the message renders as a wall of text that's hard to scan.

Concrete example (no Jira ticket):
```
*Add retry logic to webhook delivery* — https://github.com/Acme/api/pull/42

*What changed:* Added exponential backoff with 3 retries to the webhook dispatcher. Failed deliveries are now logged to the dead-letter table.

*Why:* Transient 5xx responses from partner endpoints were silently dropping events.
```

Concrete example (with Jira ticket, bug):
```
:ladybug: *[PROJ-456] Fix duplicate webhook events* — https://github.com/Acme/api/pull/99

*Ticket:* <https://example.atlassian.net/browse/PROJ-456|PROJ-456>

*What changed:* Added idempotency check in the event dispatcher using a Redis-backed dedup cache.

*Why:* Partner endpoints were receiving the same event 2-3x during peak load windows.
```

### Slack mrkdwn link-breaking rule

Slack's mrkdwn parser treats everything between `<` and the next `>` as a single link — even across line breaks. If any text (like `*Ticket:*`) sits on the next line after an unclosed or bare URL, Slack will swallow it into the link and mangle both.

**This has broken messages twice already.** The failure looks like this:
```
BAD:  ... — <https://github.com/.../pull/42
      *Ticket:>* <https://...         ← Slack ate "*Ticket:" into the URL
```

To prevent this: every line containing a URL must be followed by a **blank line**. No exceptions. The validation step below enforces this.

When sharing **multiple PRs** in a single message, separate each PR block with a blank line.

**Jira-based enhancements** (from Step 3):

- If the ticket is a **Bug**: set `<bug-prefix>` to `:ladybug: ` (with trailing space). Otherwise omit.
- If the ticket has the **release_blocker** label: insert `<release-blocker-line>` as `:rotating_light: *Release Blocker* :rotating_light:` on its own line after the title. Otherwise omit the line entirely.
- If a Jira ticket was found: insert `<ticket-line>` as `*Ticket:* <TICKET-ID link>` (e.g. `*Ticket:* <https://example.atlassian.net/browse/PROJ-123|PROJ-123>`). Otherwise omit.
- If no Jira ticket was found, omit all three.

Keep it concise and scannable for the team. Do not include the raw diff.

## Step 4.5 — Validate message before sending

Before proceeding, re-read your composed message line by line and check these rules:

1. **No broken Slack links** — every `<` that opens a link has its closing `>` on the **same line**. A `<url|label>` must never be split across lines.
2. **Blank line after every URL line** — any line containing `http://` or `https://` must be followed by a blank line (not another content line). This includes the PR URL line, the ticket URL line, and any other links.
3. **No consecutive URL lines** — two non-empty lines in a row must not both contain URLs.
4. **Labels on their own line** — `*Ticket:*`, `*What changed:*`, and `*Why:*` each start at the beginning of their own line, never appended to a URL.
5. **Section spacing** — there must be a blank line before `*What changed:*` and a blank line before `*Why:*`. These are always required for readability.
6. **Bold, not italic, for labels and title.** `*PR title*`, `*What changed:*`, `*Why:*`, `*Ticket:*`, `*Release Blocker*` must use single asterisks (`*...*` = bold in Slack mrkdwn). Single underscores (`_..._` = italic) are wrong for these — a real DM shipped with `_What changed:_` / `_Why:_` rendering as italic instead of bold. Verify every label and the title use `*` before sending.

If any rule is violated, fix the message and re-validate before continuing. Do not send a message that fails validation.

## Step 5 — Parse targets

Extract targets from the user's prompt:

- **Channel targets** — anything like `#channel-name` OR an alias / short name like "the ux channel", "post to eng", "share to design", "ping support", "the QA channel"
- **Person names** — plain names mentioned in the prompt (e.g. "send to Alex", "notify Sam and Pat")
- **Review requests** — names associated with "review" phrasing (e.g. "request review from Alex", "request their review")

If channels are mentioned:

1. **Invoke the `slack-channels` skill** to resolve each channel target. It handles alias matching, ambiguity prompts, and bootstrapping the directory if missing. Pass the user's channel reference (alias, partial name, or `#full-name`) and receive back the canonical Channel ID and channel name.
2. **If multiple matches surfaced by `slack-channels`:** stop and ask the user to disambiguate. Do not guess — wrong-channel posts are hard to undo.
3. **If no match:** the `slack-channels` skill will offer to look up via `slack_search_channels` and save the channel. Follow its prompts.

If people are mentioned:

1. **Invoke the `team-ref` skill** to resolve each person. It handles name/alias matching against the local team directory and bootstraps the directory if missing. Pass the user's reference (first name, alias, full name) and receive back the person's Slack ID, GitHub handle, and other identifiers.
2. **If a person isn't in the directory:** the `team-ref` skill will guide adding them. Follow its prompts before continuing.

If no targets at all (no channels, no people) — ask the user which channel to send to.

## Step 6 — Preview all actions

Show the user a clear summary of everything that will happen:

- **Channel posts** (if any): which channel, message preview
- **DMs** (if any): who will receive a DM, message preview
- **Review requests** (if any): who will be added as a GitHub reviewer

**Wait for user approval before proceeding.** Sending messages and requesting reviews is visible to others and irreversible.

## Step 7 — Deliver to Slack

**Channel posts:** Use `slack_send_message` MCP tool with the **Channel ID** resolved in Step 5 from `slack-channels`.

**DMs:** Use `slack_send_message` MCP tool with the person's **Slack ID as the `channel_id`**. Personalize the DM with one of two greetings, followed by a **blank line**, then the Step 4 PR block verbatim (same `*bold*` labels and blank-line spacing).

- Review requested → greeting is `Hey <first name>, here's a PR for your review:`
- Just notifying → greeting is `Hey <first name>, check out this PR:`

Concrete example — review-requested DM, fully rendered, exactly as it should appear in Slack:

```
Hey Alex, here's a PR for your review:

*refactor: replace data fetching with TanStack Query* — https://github.com/Acme/web/pull/3938

*What changed:* Migrated `app/admin/integrations/accounts/edit-account.jsx` to TanStack Query. New `queries.ts` colocates `useConnectorTypes` / `useAccountMetadata` / `useInitOAuth` / `useSaveConnector` (8 unit tests).

*Why:* Heads up — not related to your SFTP changes. While reviewing #3918 I noticed the file was a good cleanup target, so I bundled this on top of your branch.
```

Note the blank lines: greeting → title, title/URL → `*What changed:*`, `*What changed:*` → `*Why:*`. Note bold labels use single `*`, not `_`. Run the Step 4.5 checklist on the final DM string before calling `slack_send_message` — the greeting line counts as a content line, so it needs its own blank line below it.

**Skip with warning** if a person's Slack ID is `—` (or missing) in the team directory.

## Step 8 — Request GitHub reviews

If any review requests were identified:

- Run a single command: `gh pr edit <number> --add-reviewer <handle1>,<handle2>`
- **Skip with warning** if a person's GitHub handle is `—` (or missing) in the team directory.

## Tool Usage

- Use `gh` CLI for all GitHub/PR operations (do NOT use GitHub MCP tools).
- Use Slack MCP tools (`slack_search_channels`, `slack_send_message`, `slack_send_message_draft`) for all Slack operations.
- Use Atlassian MCP tools (`getJiraIssue`) for Jira lookups, via the `jira` skill if available.
