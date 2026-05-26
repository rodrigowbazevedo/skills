# Bootstrap: `slack-channels` data file

The `slack-channels` skill reads its directory from:

```
~/.claude/skills-data/slack-channels/channels.md
```

This file is **local-only** — it never ships with the skill, and reinstalling via `npx skills add` does not touch it. The first time you trigger `slack-channels` on a new machine, that file won't exist; this document walks through populating it.

## Schema

The file is a single markdown table. Columns:

| Column | Purpose | Format |
|---|---|---|
| `Channel Name` | The canonical `#handle` as it appears in Slack | Always prefixed with `#`, e.g. `#user-experience` |
| `Channel ID` | The Slack-internal stable identifier | Starts with `C` (public/private), `D` (DM), or `G` (group), 9–11 alphanumerics |
| `Aliases` | Comma-separated short names you use in conversation | Lowercase recommended, no `#`, e.g. `ux, user-experience, design` |
| `Purpose` | One-line description of what the channel is for | Free text; use `—` if unknown |

Rows are sorted alphabetically by **Channel Name**.

Use `—` (em-dash) for empty fields — never leave a cell blank, because that breaks the markdown table.

## Empty starter template

Copy everything between the fences below into `~/.claude/skills-data/slack-channels/channels.md`. The placeholder row is illustrative — delete it once you've added real entries.

```markdown
| Channel Name | Channel ID | Aliases | Purpose |
|--------------|------------|---------|---------|
| #example | C00000000 | example, sample | Placeholder — replace with real channels |
```

## Path A — Manual edit

For a small number of channels (~5 or fewer), or when you don't have the Slack MCP installed:

1. Create the directory: `mkdir -p ~/.claude/skills-data/slack-channels`
2. Open `~/.claude/skills-data/slack-channels/channels.md` in your editor.
3. Paste the empty template above.
4. For each channel you care about:
   - Open Slack, navigate to the channel.
   - Click the channel name to open the details panel.
   - At the bottom, find **Channel ID** (e.g. `C0XXXXXXXXX`). Copy it.
   - Add a row with the channel's full handle, the ID, your preferred aliases, and a one-line purpose.
5. Keep rows sorted alphabetically by Channel Name.

## Path B — Interactive seed (Slack MCP)

If the **Slack MCP** is installed, run workflow 5 in `SKILL.md` (or simply say "seed channels" / "discover channels" / "set up the channel directory"). The skill will:

1. Call `slack_search_channels` with a broad query (or empty/wildcard) to list channels you're a member of.
2. Optionally run additional searches for prefixes your workspace uses (e.g. `eng-`, `team-`, `proj-`).
3. Show the candidate list with channel name, ID, and topic/member count.
4. Ask which to index. A reasonable default: channels with > 5 members and an active topic.
5. For each picked channel, ask for aliases (with sensible defaults derived from the name) and a one-line purpose.
6. Write all rows to `~/.claude/skills-data/slack-channels/channels.md` in alphabetical order.

### Private channels

`slack_search_channels` may not surface every private channel — Slack's API enforces visibility. If you need a private channel in the directory and it doesn't appear in seed results, add it manually (Path A) or use workflow 3 ("add a channel") with the full `#handle`; the MCP can usually resolve the ID once you name the channel directly.

## Path C — Import from another machine

If you already have a `channels.md` from another machine or a backup:

1. `mkdir -p ~/.claude/skills-data/slack-channels`
2. Copy the file: `cp /path/to/existing/channels.md ~/.claude/skills-data/slack-channels/channels.md`
3. Open it and re-check for stale entries — channel IDs are stable, but channels you've left or that have been archived are worth pruning.

## After bootstrap

Trigger a quick lookup to confirm the file is wired up — e.g. "what's the channel for X" or "list channels". The skill should read the file and respond with the table. If it reports the file as missing, double-check the path (`~/.claude/skills-data/slack-channels/channels.md`) — the SKILL.md hardcodes this path.

## Updating later

To add a channel later, use workflow 3 in `SKILL.md` ("add the X channel as ux,user-experience") — it will append a sorted row.

To remove a channel, use workflow 4 ("forget the X channel" / "remove the X channel") — it will delete the row.

To edit a row in place (rename aliases, fix the purpose), just open the file in your editor — the table is plain markdown and matching is case-insensitive, so don't worry about capitalization.
