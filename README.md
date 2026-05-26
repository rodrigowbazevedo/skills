# skills

Personal [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skills, installable via the [`skills` CLI](https://github.com/vercel-labs/skills).

## Install

All skills, globally:

```bash
npx skills add -g rodrigowbazevedo/skills --all
```

A single skill:

```bash
npx skills add -g rodrigowbazevedo/skills -s typescript
```

Project-local (drops into `./.claude/skills/`):

```bash
npx skills add rodrigowbazevedo/skills --all
```

## What's inside

| Skill | Summary |
| --- | --- |
| [`javascript`](./skills/javascript) | Personal JS style: early returns over `else`, `const` over `let`, no chained ternaries, brace all control-flow bodies, `for…of` over `.forEach`, generators for lazy iteration. Auto-triggers on `.js` / `.jsx` edits. |
| [`typescript`](./skills/typescript) | TS rules layered on `javascript`: don't lie to the compiler (no `as` / `any` / `!`), let the type system do the work (generic inference, `satisfies`, object enums, discriminated unions with `never` exhaustiveness). Auto-triggers on `.ts` / `.tsx` edits. |
| [`my-prs`](./skills/my-prs) | Lists your open PRs across Claravine (TrackingFirst) repos, grouped by review status (approved / changes requested / waiting). Trigger with "list my PRs", "my review queue", etc. |
| [`plan-resolve-questions`](./skills/plan-resolve-questions) | Runs after every `ExitPlanMode` call: if the plan has unresolved questions, presents each one to you via `AskUserQuestion`. Overrides the "don't pause for clarifying questions" auto-mode heuristic. |
| [`plan-to-github-issue`](./skills/plan-to-github-issue) | Runs after every `ExitPlanMode` call: offers to save the plan as a GitHub issue (or update an existing one) and auto-links it into any open PR on the current branch as `Closes #N`. |

## Notes

- `my-prs` is hard-coded to Claravine / TrackingFirst repos; fork it if you want to point it elsewhere.
- `plan-resolve-questions` and `plan-to-github-issue` are designed to auto-fire after `ExitPlanMode` — pairing them with a line in your `~/.claude/CLAUDE.md` reinforces the trigger.
- These reflect personal preferences, not universal best practice — read each `SKILL.md` before installing.

## License

MIT — see [LICENSE](./LICENSE).
