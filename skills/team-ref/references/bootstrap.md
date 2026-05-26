# Bootstrap: `team-ref` data file

The `team-ref` skill reads its directory from:

```
~/.claude/skills-data/team-ref/team-members.md
```

This file is **local-only** — it never ships with the skill, and reinstalling via `npx skills add` does not touch it. The first time you trigger `team-ref` on a new machine, that file won't exist; this document walks through populating it.

## Schema

The file is a single markdown table. Columns:

| Column | Purpose | Format |
|---|---|---|
| `Name` | Full display name | `First Last` |
| `Email` | Work email | Standard email format |
| `Slack ID` | The person's stable Slack user ID | Starts with `U` (humans) or `W` (Enterprise Grid users), 9–11 alphanumerics |
| `GitHub` | GitHub username | No `@` prefix |
| `Jira Account ID` | Atlassian account ID | Format `123456:UUID` or a hex string, depending on your Jira deployment |
| `Title` | Job title | Free text; `—` if unknown |
| `Aliases` | Comma-separated nicknames/short names | Used for matching; `—` if none |

Rows are sorted alphabetically by first name (i.e. by the **Name** column).

Use `—` (em-dash) for empty fields — never leave a cell blank, because that breaks the markdown table.

## Empty starter template

Copy everything between the fences below into `~/.claude/skills-data/team-ref/team-members.md`. The placeholder row is illustrative — delete it once you've added real entries.

```markdown
| Name | Email | Slack ID | GitHub | Jira Account ID | Title | Aliases |
|------|-------|----------|--------|-----------------|-------|---------|
| Alex Example | alex@example.com | U00000000 | alexexample | 000000:00000000-0000-0000-0000-000000000000 | Senior Engineer | Al, Lex |
```

## Path A — Manual edit

For a small team (~10 or fewer people), or when you don't have the Slack/GitHub/Atlassian MCPs installed:

1. Create the directory: `mkdir -p ~/.claude/skills-data/team-ref`
2. Open `~/.claude/skills-data/team-ref/team-members.md` in your editor.
3. Paste the empty template above.
4. For each colleague you want to track, gather their identifiers:
   - **Slack ID**: in Slack, click the person's name → "View full profile" → click `⋮` → **Copy member ID**.
   - **GitHub**: their `github.com/<username>` slug.
   - **Jira Account ID**: open any ticket they're assigned to → click their avatar → URL shows `accountId=<...>`. Or use `lookupJiraAccountId` if you have the Atlassian MCP.
   - **Email**: their work email — usually visible in Slack profile or your contacts.
   - **Title**: optional; Slack profile or LinkedIn.
   - **Aliases**: any short names you actually use when referring to them (e.g. nicknames, initials).
5. Add a row per person. Keep them sorted alphabetically by first name.
6. Use `—` for anything you can't fill in.

## Path B — Interactive add-by-add (with available MCPs)

If you have any of these installed, run workflow 2 in `SKILL.md` (or simply say "add Alex to the team directory" / "add team member"). The skill will:

1. Try to gather identifiers from whichever sources are available:
   - **Slack MCP**: `slack_search_users` by name or email.
   - **`gh` CLI**: `gh api /orgs/<your-org>/members` and match by name/username. (The skill will ask you for the GitHub org if it doesn't already know — or you can supply the handle directly.)
   - **Atlassian MCP**: `lookupJiraAccountId` with the email.
2. Ask you for anything it couldn't resolve.
3. Append the row to the data file in alphabetical order.
4. Show you what was saved.

Repeat for each team member you want indexed. There's no batch mode — each person is one round-trip.

### Tips for fast bulk-add

- Get the Slack export of your team workspace if you have admin access — it includes member IDs and emails in one place.
- For a GitHub org you administer, `gh api /orgs/<org>/members --paginate` returns every member; pair each with their Slack ID by email.
- For Jira, `getAccessibleAtlassianResources` followed by user searches works, but the per-user `lookupJiraAccountId` is usually faster if you already have emails.

## Path C — Import from another machine

If you already have a `team-members.md` from another machine or a backup:

1. `mkdir -p ~/.claude/skills-data/team-ref`
2. Copy the file: `cp /path/to/existing/team-members.md ~/.claude/skills-data/team-ref/team-members.md`
3. Open it and prune anyone who's no longer on the team.

## Privacy note

The `team-members.md` file contains personal identifiers (names, emails, Slack IDs). Keep it **out of any git repository** that you push to a remote you don't control. The default location (`~/.claude/skills-data/`) is intentionally outside both the skill install path and any typical project directory.

The same goes for sharing this file — if you migrate it between machines, prefer encrypted transport (e.g. via 1Password, an SSH-based copy) rather than email or chat.

## After bootstrap

Trigger a quick lookup to confirm the file is wired up — e.g. "who is Alex" (using a real name you added). The skill should read the file and respond with the matching row. If it reports the file as missing, double-check the path (`~/.claude/skills-data/team-ref/team-members.md`) — the SKILL.md hardcodes this path.

## Updating later

To add a new colleague, use workflow 2 in `SKILL.md` — it appends a sorted row.

To remove someone who's left the team, use workflow 3 ("remove X from the team directory") — it deletes the row.

To edit a row in place (correct a typo, add an alias, update a title), just open the file in your editor.
