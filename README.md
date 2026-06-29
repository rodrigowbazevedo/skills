# skills

Personal [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skills (and cross-agent compatible — Codex, Cursor, Cline, etc.), installable via the [`skills` CLI](https://github.com/vercel-labs/skills).

## Install

```bash
npx skills add -g rodrigowbazevedo/skills --all              # all skills, global
npx skills add -g rodrigowbazevedo/skills -s typescript      # one skill, global
npx skills add rodrigowbazevedo/skills --all                 # all skills, project-local (./.claude/skills/)
```

The `-g` flag installs into `~/.agents/skills/` and symlinks them into every agent runtime on your machine; drop it to install into the current project only.

## Skills

### Code style

Personal style preferences that auto-fire whenever a JS/TS file is touched.

| Skill | Triggers | Summary |
| --- | --- | --- |
| [`javascript`](./skills/javascript) | `.js` / `.jsx` edits; "early return", "prefer const", "no forEach", "use a generator" | Early returns over `else`, `const` over `let`, no chained ternaries, always brace control-flow bodies, `for…of` over `.forEach`, generators for lazy iteration. |
| [`typescript`](./skills/typescript) | `.ts` / `.tsx` edits; "fix this type error", "narrow this type", "use satisfies", "avoid enum" | Layers on `javascript`. Don't lie to the compiler (no `as` / `any` / `!` / `@ts-ignore`) and let the type system do the work (generic inference, `satisfies`, `as const`, object enums, discriminated unions with `never` exhaustiveness). |

> The two skills are designed to compose: loading `typescript` implicitly requires `javascript`, so both rule sets apply to `.ts` / `.tsx` work.

### Plan-mode workflow

Hooks that auto-fire after every `ExitPlanMode` call to make plan-mode less manual.

| Skill | Triggers | Summary |
| --- | --- | --- |
| [`plan-resolve-questions`](./skills/plan-resolve-questions) | Auto-fires after `ExitPlanMode` when the plan has unresolved questions | Presents each unresolved question to you via `AskUserQuestion`. Overrides the "don't pause for clarifying questions" auto-mode heuristic. |
| [`plan-to-github-issue`](./skills/plan-to-github-issue) | Auto-fires after `ExitPlanMode` in any repo with a GitHub remote | Defers (no blocking prompt) so implementation starts immediately, then asks to save the plan as a GitHub issue (or update an existing one) right before the first commit/PR, and auto-links it into any open PR on the current branch as `Closes #N`. |

> Pairing with a line in `~/.claude/CLAUDE.md` like *"After every `ExitPlanMode` call, invoke `plan-resolve-questions` then `plan-to-github-issue`"* reinforces the auto-fire and makes the behavior survive description-matching drift.

### PR workflow

| Skill | Triggers | Summary |
| --- | --- | --- |
| [`my-prs`](./skills/my-prs) | "list my PRs", "my review queue", "PRs waiting on me" | Lists your open PRs across [Claravine (TrackingFirst)](https://github.com/TrackingFirst) repos, grouped by review status (approved / changes requested / waiting on reviewer). |
| [`share-pr`](./skills/share-pr) | "share PR", "send PR to Slack", "DM \<person\> about PR", "request review from \<person\>" | Composes a Slack-formatted message from the current PR (title, diff context, optional Jira ticket), then posts to channels, DMs people, and requests GitHub reviews. Delegates channel/person resolution to `slack-channels` and `team-ref`. |

> `my-prs` is hard-coded to the TrackingFirst org — fork it and swap the org if you want to point it elsewhere. `share-pr` is org-agnostic but needs the Slack MCP installed; the Jira block activates only if the `jira` skill and Atlassian MCP are also present.

### Slack & team lookups

Local directories of channels and colleagues. The skills' *logic* is published here, but the **data lives in `~/.claude/skills-data/`**, outside the install path — your personal mappings are never committed. Each skill detects a missing data file and walks you through bootstrap on first use.

| Skill | Triggers | Summary |
| --- | --- | --- |
| [`slack-channels`](./skills/slack-channels) | "send to the ux channel", "post in eng", "list channels", "add this channel as ux", "seed channels" | Resolves channel aliases ("the ux channel", "support") into canonical `#handle` + Slack channel ID. Add/remove/seed workflows mutate the local directory. |
| [`team-ref`](./skills/team-ref) | "who is X", "what's X's Slack/GitHub/Jira", "add team member", "remove X from the team directory" | Resolves people across Slack/GitHub/Jira from a local table. Lookup is offline-only; add-member can use Slack/GitHub/Atlassian MCPs if available. |

> Bootstrap docs: [`slack-channels/references/bootstrap.md`](./skills/slack-channels/references/bootstrap.md) and [`team-ref/references/bootstrap.md`](./skills/team-ref/references/bootstrap.md). Both include schema + empty starter table + manual and interactive seed paths.

## Authoring

See [CLAUDE.md](./CLAUDE.md) for the skill-authoring guide (frontmatter format, the 1024-char description limit, token-efficient layout, and the "add a new skill" checklist).

## Compatibility

Installed via `npx skills add -g`, these skills surface universally across the agents the `skills` CLI supports today — Amp, Antigravity, Cline, Codex, Cursor, Roo Code, Continue, Goose, Q Developer, OpenCode, Trae, plus symlinked drops into AiderDesk, Augment, Claude Code, OpenClaw, and ~35 more.

In practice they're authored against Claude Code's runtime (skills, plan mode, `AskUserQuestion`, the `Skill` tool); other agents will read the markdown but may not have the same primitives — YMMV.

## Notes

- These reflect personal preferences, not universal best practice — read each `SKILL.md` before installing.
- The `plan-*` skills assume Claude Code's plan-mode UX; they're no-ops elsewhere.
- The Slack/team-lookup skills store their data outside the install path (`~/.claude/skills-data/`) so reinstalling never touches your local tables. On a fresh machine, the first invocation guides you through bootstrap.
- Some descriptions are quite long because they intentionally enumerate trigger phrases for reliable activation. The 1024-char cap still applies — see `CLAUDE.md`.

## License

MIT — see [LICENSE](./LICENSE).
