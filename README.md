# engineering-standards

House engineering, UX, and design guidelines for my projects — written as a
context file that AI coding agents actually read on every turn.

## What's here

- **[AGENTS.template.md](AGENTS.template.md)** — the template. Copy it into a new
  repo, delete the stack modules you don't need, fill in the project brief.

## Using it in a new repo

```sh
# from the new repo's root
curl -sO https://raw.githubusercontent.com/Anoop-Goudar/engineering-standards/main/AGENTS.template.md
mv AGENTS.template.md AGENTS.md
printf '@AGENTS.md\n' > CLAUDE.md
```

Then:

1. Delete every **Stack Module** in Part 2 except the one you're building.
2. Fill in **Part 5 — Project brief**. It's the only section that should differ
   between projects.
3. Delete the `HOW TO USE` comment block at the top.

`CLAUDE.md` is a one-line pointer so Claude Code and other agents read the same
file. Don't maintain two copies.

## Why it's structured this way

The file is injected into the agent's context on **every turn**, so length isn't
free — a long rulebook gets skimmed and diluted. Anything a linter, a type, or CI
can enforce belongs there, not in prose. Anything long belongs in `docs/`
(`ARCHITECTURE.md`, `DESIGN.md`, `BACKLOG.md`), which is read on demand.

The layering matters:

| Part | Scope |
|---|---|
| 1 — Universal | Only rules that hold across *every* project. Settled service choices live here. |
| 2 — Stack modules | Pick one, delete the rest. Keeps one project's stack from becoming universal law. |
| 3 — UX standards | Web. Skipped entirely by the mobile module. |
| 4 — Design defaults | **Defaults, not law.** A project's `docs/DESIGN.md` overrides them. |
| 5 — Project brief | The only per-project section. |

Part 1 is deliberately small. Everything in it earned its place by being true in
practice across the existing repos — not by sounding correct.

## Keeping it honest

When a rule here is consistently wrong for a real project, change it *here*,
deliberately — don't let the projects quietly drift. If a new project's stack
doesn't fit any module, add a module rather than bending Part 1.
