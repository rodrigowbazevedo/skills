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
| [`plan-to-github-issue`](./skills/plan-to-github-issue) | Auto-fires after `ExitPlanMode` in any repo with a GitHub remote | Offers to save the plan as a GitHub issue (or update an existing one) and auto-links it into any open PR on the current branch as `Closes #N`. |

> Pairing with a line in `~/.claude/CLAUDE.md` like *"After every `ExitPlanMode` call, invoke `plan-resolve-questions` then `plan-to-github-issue`"* reinforces the auto-fire and makes the behavior survive description-matching drift.

### GitHub workflow

| Skill | Triggers | Summary |
| --- | --- | --- |
| [`my-prs`](./skills/my-prs) | "list my PRs", "my review queue", "PRs waiting on me" | Lists your open PRs across [Claravine (TrackingFirst)](https://github.com/TrackingFirst) repos, grouped by review status (approved / changes requested / waiting on reviewer). |

> `my-prs` is hard-coded to the TrackingFirst org — fork it and swap the org if you want to point it elsewhere.

## Authoring

See [CLAUDE.md](./CLAUDE.md) for the skill-authoring guide (frontmatter format, the 1024-char description limit, token-efficient layout, and the "add a new skill" checklist).

## Compatibility

Installed via `npx skills add -g`, these skills surface universally across the agents the `skills` CLI supports today — Amp, Antigravity, Cline, Codex, Cursor, Roo Code, Continue, Goose, Q Developer, OpenCode, Trae, plus symlinked drops into AiderDesk, Augment, Claude Code, OpenClaw, and ~35 more.

In practice they're authored against Claude Code's runtime (skills, plan mode, `AskUserQuestion`, the `Skill` tool); other agents will read the markdown but may not have the same primitives — YMMV.

## Notes

- These reflect personal preferences, not universal best practice — read each `SKILL.md` before installing.
- The `plan-*` skills assume Claude Code's plan-mode UX; they're no-ops elsewhere.
- Some descriptions are quite long because they intentionally enumerate trigger phrases for reliable activation. The 1024-char cap still applies — see `CLAUDE.md`.

## License

MIT — see [LICENSE](./LICENSE).
