---
name: team-ref
description: >
  Personal team directory — look up colleagues' email, Slack, GitHub, and Jira
  identifiers. Use when the user asks "who is X", "what's X's Slack/GitHub/Jira",
  "find team member", "tag X", "mention X", "assign to X", needs to resolve
  a person's handle across platforms, or wants to "add/remove team member",
  "update team directory", "sync team list".
---

# Team Reference

Personal team directory for resolving people across Slack, GitHub, and Jira.

## Prerequisites

- **None required for lookup.** The default workflow reads a local markdown file — no MCPs, no network.
- **Optional MCPs for bootstrap and add-member workflows** — if installed, they speed up populating the directory:
  - **Slack MCP** (`slack_search_users`) — discover Slack IDs and emails.
  - **`gh` CLI** — discover GitHub handles via `gh api /orgs/<your-org>/members`.
  - **Atlassian MCP** (`lookupJiraAccountId`) — resolve Jira account IDs from email.

  None of these are required — the directory can be populated entirely by hand.

## Data file

The directory is stored at:

```
~/.claude/skills-data/team-ref/team-members.md
```

This path is **outside** any installed skill directory, so reinstalling the skill via `npx skills add` doesn't touch your data. The path is hardcoded throughout this skill — if you relocate the file, also update the references below.

## Preflight — before any workflow

Before any lookup, add, or remove operation, check whether `~/.claude/skills-data/team-ref/team-members.md` exists:

- **If present:** proceed to the requested workflow.
- **If missing:** do not attempt lookups against a non-existent file. Instead, read `references/bootstrap.md` and offer the user three paths:
  1. **Paste-and-edit**: open the empty template from `references/bootstrap.md`, fill in by hand, save to the data file path.
  2. **Interactive add-by-add**: use workflow 2 below to add team members one at a time, optionally sourcing identifiers from Slack/GitHub/Jira MCPs if available.
  3. **Import existing**: if the user already has a `team-members.md` from another machine, ask them to paste it or supply a path; copy it to the data file path.

After bootstrap, create the directory (`mkdir -p ~/.claude/skills-data/team-ref`) if needed and write the file. Then continue with the originally requested workflow.

## Workflows

### 1. Lookup (default)

Read `~/.claude/skills-data/team-ref/team-members.md` and find the matching person by name, email, handle, or any partial match. Return their full row of identifiers. If multiple matches, list all. If no match, say so plainly.

Matching is case-insensitive and checks both the **Name** and **Aliases** columns.

### 2. Add a new team member

When the user asks to add someone:

1. Gather identifiers from available sources (skip any whose MCP/CLI isn't installed):
   - **Slack**: search by name or email using `slack_search_users`.
   - **GitHub**: `gh api /orgs/<your-org>/members` and match by name/username. Replace `<your-org>` with the GitHub org you care about; if your team isn't in a single org, ask the user for the handle directly.
   - **Jira**: use `lookupJiraAccountId` with their email.
2. Any field you can't resolve, ask the user for or set to `—`.
3. Append a new row to `~/.claude/skills-data/team-ref/team-members.md` in alphabetical order by first name.
4. Confirm the addition showing what was found and what was left as `—`.

### 3. Remove a departed team member

When the user asks to remove someone:

1. Find the person in `~/.claude/skills-data/team-ref/team-members.md`.
2. Remove their row.
3. Confirm the removal.

## Editing the table by hand

`~/.claude/skills-data/team-ref/team-members.md` is plain markdown. The user can edit it directly. The table format is:

```
| Name | Email | Slack ID | GitHub | Jira Account ID | Title | Aliases |
|------|-------|----------|--------|-----------------|-------|---------|
| Alex Example | alex@example.com | U00000000 | alexexample | 000000:00000000-0000-0000-0000-000000000000 | Senior Engineer | Al, Lex |
```

Aliases are comma-separated, lowercase recommended (matching is case-insensitive anyway). Use `—` for empty fields, never blank, so the table stays well-formed.

## How `share-pr` (and other workflows) use this skill

`share-pr` parses person names from the user's prompt and **invokes this skill** to resolve each name to a Slack ID (for DMs) and a GitHub handle (for review requests). If a person isn't in the directory, this skill's add workflow (2) is offered before continuing.

## Reference Index

| File | Purpose | When to read |
|------|---------|--------------|
| `references/bootstrap.md` | Empty starter template + manual and interactive setup instructions | When Preflight detects a missing data file |
| `~/.claude/skills-data/team-ref/team-members.md` | The team directory itself (local, never committed) | Every lookup, add, or remove operation |
