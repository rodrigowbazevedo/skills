---
name: slack-channels
description: >
  Personal Slack channel directory — resolves channel aliases ("the ux channel",
  "eng", "design", "support") into canonical channel names (e.g.
  `#user-experience`) and Slack channel IDs. Use this skill BEFORE any Slack
  send action (share-pr, slack_send_message, posting an announcement) whenever
  the user names a channel by alias, nickname, short name, team name, or
  partial match instead of the full `#handle`. Triggers on phrases like
  "send to the ux channel", "post in eng", "ping the design channel",
  "share to support", "tell the QA channel", as well as "add channel",
  "remove channel", "list channels", "what's the channel for X",
  "seed channels", "index this channel". Do NOT skip this skill just
  because the user named the channel without a leading `#` — alias
  resolution is exactly its job, and getting the wrong channel posts
  a message to the wrong audience.
metadata:
  mcp-server: slack
---

# Slack Channels

Personal Slack channel directory. Resolves short aliases into canonical channel names and IDs so messages don't have to spell out long handles every time.

## Prerequisites

- **Slack MCP** (required for the add/seed workflows) — uses `slack_search_channels` to discover channels in your workspace and resolve `#full-name` references to IDs.
- **Lookup workflow (1)** does not require Slack MCP — it reads only the local directory file.

## Data file

The directory is stored at:

```
~/.claude/skills-data/slack-channels/channels.md
```

This path is **outside** any installed skill directory, so reinstalling the skill via `npx skills add` doesn't touch your data. The path is hardcoded throughout this skill — if you relocate the file, also update the references below.

## Preflight — before any workflow

Before any lookup, list, add, remove, or seed operation, check whether `~/.claude/skills-data/slack-channels/channels.md` exists:

- **If present:** proceed to the requested workflow.
- **If missing:** do not attempt lookups against a non-existent file. Instead, read `references/bootstrap.md` and offer the user three paths:
  1. **Paste-and-edit**: open the empty template from `references/bootstrap.md`, fill in by hand, save to the data file path.
  2. **Interactive seed**: run workflow 5 below — calls `slack_search_channels` to discover channels in the workspace and walks the user through aliases + purposes.
  3. **Import existing**: if the user already has a `channels.md` from another machine, ask them to paste it or supply a path; copy it to the data file path.

After bootstrap, create the directory (`mkdir -p ~/.claude/skills-data/slack-channels`) if needed and write the file. Then continue with the originally requested workflow.

## Why this skill exists

Slack channel handles are often long and prefixed. Typing them out is friction, and getting one wrong sends a message to the wrong audience. This skill lets the user say "the ux channel" and have the right channel resolve deterministically. The directory is the source of truth — when in doubt, read it.

## Workflows

### 1. Lookup / resolve (default)

When the user references a channel by anything other than an exact `#full-name` already in the directory:

1. Read `~/.claude/skills-data/slack-channels/channels.md`.
2. Match the user's reference (case-insensitively) in this priority order:
   - **Aliases** column — exact or substring match. This is the most reliable signal because aliases are user-curated.
   - **Channel Name** column — substring match (e.g. "ux" matches `#user-experience`).
   - **Purpose** column — keyword match. Last resort, since purpose is freeform.
3. Return the resolved row: full channel name, channel ID, and one-line purpose.
4. **If multiple matches:** list them all and ask the user to disambiguate. Don't guess.
5. **If no match:** say so plainly. Offer to look up via `slack_search_channels` and add it (workflow 3).

If the user already gave an exact `#full-name`:
- If it exists in the directory, return its ID directly.
- If it's not in the directory, fetch the ID via `slack_search_channels` so the send can proceed, and offer to save it for next time (workflow 3).

The Channel ID is what downstream tools (`slack_send_message`) actually need — names can be renamed in Slack, IDs can't.

### 2. List / browse

When the user asks "what channels do I have indexed?", "list channels", "show my channels": print the full table from `~/.claude/skills-data/slack-channels/channels.md`. If the table is empty (or the file doesn't exist), trigger Preflight to bootstrap or suggest workflow 5 (seed).

### 3. Add a channel

When the user asks to add or save a channel (e.g. "add the ux channel as ux", "save #user-experience as ux,user-experience"):

1. Resolve the canonical channel name and ID:
   - If the user gave a `#full-name`, call `slack_search_channels` with that name to get the ID.
   - If the user gave an alias only (and it isn't already in the directory), ask which actual channel they mean and search for it.
2. Ask for any additional aliases they want (comma-separated). Suggest sensible defaults from the channel name (e.g. for `#user-experience` → `ux, user-experience`).
3. Ask for a one-line purpose. If the channel has a topic/description from `slack_search_channels`, suggest that as a starting point.
4. Append a new row to `~/.claude/skills-data/slack-channels/channels.md`, keeping the table sorted alphabetically by **Channel Name**.
5. Confirm what was saved by showing the new row.

### 4. Remove a channel

When the user asks to remove, forget, or unindex a channel:

1. Resolve the row via workflow 1.
2. Remove the row from `~/.claude/skills-data/slack-channels/channels.md`.
3. Confirm what was removed.

### 5. Seed / discover (first-time setup or refresh)

When the user asks to "seed channels", "discover channels", "set up the channel directory", or the data file is empty/missing:

1. Call `slack_search_channels` with a broad query (or empty/wildcard) to list channels the user is a member of. If that's not enough, run additional searches for likely prefixes the user's workspace uses.
2. Show the candidate list to the user with one channel per line: channel name, ID, member count or topic if available.
3. Ask the user which to index. A sensible default suggestion: their member channels with > 5 members and an active topic.
4. For each picked channel, ask for aliases (suggest defaults derived from the name) and a purpose.
5. Append all rows to `~/.claude/skills-data/slack-channels/channels.md` in alphabetical order.
6. Confirm with the count and a preview of the table.

## How `share-pr` (and other senders) use this skill

`share-pr` parses channel targets from the user's prompt and **invokes this skill** to resolve each alias to a Channel ID before calling `slack_send_message`. If a target doesn't resolve cleanly, surface the ambiguity to the user — never silently guess. Posting to the wrong channel is hard to undo.

## Editing the table by hand

`~/.claude/skills-data/slack-channels/channels.md` is plain markdown. The user can edit it directly — add aliases, fix typos, reorder rows. The table format is:

```
| Channel Name | Channel ID | Aliases | Purpose |
|--------------|------------|---------|---------|
| #user-experience | C0XXXXXXXXX | ux, user-experience | Internal UX team coordination |
```

Aliases are comma-separated, lowercase recommended (matching is case-insensitive anyway). Use `—` for empty fields, never blank, so the table stays well-formed.

## Reference Index

| File | Purpose | When to read |
|------|---------|--------------|
| `references/bootstrap.md` | Empty starter template + manual and interactive seed instructions | When Preflight detects a missing data file |
| `~/.claude/skills-data/slack-channels/channels.md` | The channel directory itself (local, never committed) | Every lookup, list, add, remove, or seed operation |
