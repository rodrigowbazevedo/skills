# Agent Guide: skills

Adapted from [TrackingFirst/agent-skills `AGENTS.md`](https://github.com/TrackingFirst/agent-skills/blob/main/AGENTS.md), trimmed for a small personal repo.

## What this repo is

A small collection of personal **skills** — structured prompt files that teach AI agents (Claude Code, Cursor, Codex, etc.) how to perform specific, repeatable tasks. Skills are installed into a target environment via `npx skills add`, after which the agent can invoke them by name or trigger phrase.

## Directory layout

```
skills/                        # One folder per skill
  <skill-name>/
    SKILL.md                   # Required — skill definition (frontmatter + instructions)
    references/                # Optional — reference docs the skill reads at runtime
    scripts/                   # Optional — shell scripts the skill may execute
    assets/                    # Optional — static assets (images, templates)
.claude-plugin/
  plugin.json                  # Lists the skills published from this repo
README.md                      # Human-readable skills index
```

## Skill anatomy

Every skill is defined by a `SKILL.md` with YAML frontmatter followed by Markdown instructions:

```markdown
---
name: my-skill
description: What the skill does and when the agent should invoke it.
---

# My Skill

Instructions for the agent go here.
```

### Frontmatter fields

| Field | Required | Purpose |
|---|---|---|
| `name` | Yes | Unique identifier; must match the folder name |
| `description` | Yes | One-paragraph summary + trigger phrases. **This is what the agent reads to decide when to invoke the skill.** Hard limit: **1024 characters** — skills with longer descriptions fail to load. |

> **Description length limit:** the `description` is hard-capped at **1024 characters**; over the limit and the skill won't load. Focus the description on *when* to invoke (triggers, symptoms, situations), not the full workflow — the body of `SKILL.md` is where the how-to lives. Folded YAML scalars (`description: >`) collapse newlines to spaces, so count post-fold.

## How triggering works

When you type a message, the agent scans the `description` of every installed skill. If the description matches your intent, the agent loads the full `SKILL.md` and follows its instructions. Keep descriptions precise — include explicit trigger phrases for reliable activation.

## Conventions

- Folder names: `lowercase-hyphen` (e.g. `plan-to-github-issue`, `my-prs`).
- `SKILL.md` should stay under ~200 lines of core instructions — extract the rest into `references/`.
- One skill per folder; no nested skill folders.
- If you add or remove a skill, update **both** `README.md` (the skills table) and `.claude-plugin/plugin.json` (the `skills` array).

### Token-efficient skill design

`SKILL.md` is **always fully loaded** when triggered — every line costs tokens. Reference files in `references/` are loaded **on-demand**, only when a specific workflow needs them.

**When to split:** extract a section into `references/` if it's >50 lines of reference material or only needed for a specific sub-workflow.

**How to reference:** point at the file from `SKILL.md` with explicit guidance on *when* to read it. Add a **Reference Index** at the bottom of `SKILL.md`:

```markdown
## Reference Index

| File | Purpose | When to read |
|------|---------|--------------|
| `references/foo.md` | Detailed Foo workflow | Only when the user asks to Foo |
```

## Skill creation workflow

If you have the [`skill-creator`](https://github.com/anthropics/skills) skill installed, invoke it with `/skill-creator` for drafting and iterating on new skills — it handles description tuning and basic eval.

Otherwise, work from the minimal template below.

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md` with `name` + `description` frontmatter and instructions.
2. If `SKILL.md` exceeds ~200 lines, extract reference material into `references/` and add a Reference Index table.
3. If needed, add supporting files under `scripts/` or `assets/`.
4. Add the skill to the table in `README.md`.
5. Add the skill path to the `skills` array in `.claude-plugin/plugin.json`.
6. Commit and push to `main`.

Minimal template:

```markdown
---
name: my-skill
description: What the skill does and when to use it. Trigger phrases: "do the thing", "run my-skill".
---

# My Skill

Instructions for the agent go here.
```

## Install

Consumers install via:

```bash
npx skills add -g rodrigowbazevedo/skills --all              # all, global
npx skills add -g rodrigowbazevedo/skills -s typescript      # one skill
npx skills add rodrigowbazevedo/skills --all                 # project-local
```

See [README.md](README.md) for the full skills list.
