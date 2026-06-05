# Skills — on-demand playbooks for the employee

A skill is a reusable playbook the **employee** loads when a task matches it —
the same progressive-disclosure idea as this meta-harness skill, but shipped
inside the bundle to shape the *employee's* behavior.

Only the skill's `name — description` sits in the employee's system prompt (cheap).
The **body is loaded on demand** via the `skill` tool when the model judges a
skill applies. So you can ship deep, detailed playbooks without paying their
context cost every turn.

## Layout (this is the part docs get wrong)

A skill is a **directory with a `SKILL.md`**, not a flat file:

```
skills/
├── triage-inbox/
│   └── SKILL.md
└── draft-quote/
    ├── SKILL.md
    └── reference/
        └── pricing-table.md      # optional on-demand depth the SKILL.md points to
```

`SKILL.md` needs YAML frontmatter with a **required `description`** (the trigger)
and an optional `name` (defaults to the directory name):

```markdown
---
name: triage-inbox
description: Use when a new email arrives and needs sorting into urgent / later / ignore.
---

# Triage the inbox

1. Read the latest unread thread.
2. Classify: urgent (customer blocked), later (FYI), ignore (newsletter).
3. For urgent, draft a holding reply and flag the human.
```

Rules the runtime enforces:
- A skill **must** have `SKILL.md` with a `description`, or it's skipped with a
  warning (never silently half-loaded).
- Directories starting with `.` or `_` are ignored.
- Skill names match `^[a-zA-Z0-9][a-zA-Z0-9._-]*$`.

> **Do not** write `skills/triage-inbox.md` (a flat file). Older docs show that
> shape; the runtime requires `skills/<name>/SKILL.md`.

## How it's used at runtime

1. At session start the platform scans `skills/`, reads each frontmatter, and
   injects an `## Available Skills` catalog (just `name — description`) into the
   employee's prompt — with an explicit note that the bodies are *not* present.
2. When a request matches, the employee calls the `skill` tool with the name; the
   tool returns the full `SKILL.md` body, which the model treats as authoritative
   for that turn. `reference/` files inside a skill are loaded only if the body
   points to them.

## Authoring guidance

- Write the **`description` as a trigger** ("Use when …"), not a summary — it's
  the only thing the model sees before deciding to load.
- Keep the body a **procedure**, not prose. Steps, decision tables, exact commands.
- Push depth into `reference/` files the body links to (progressive disclosure all
  the way down).
- Same shape as this skill you're reading — mirror it.

## Author does vs. platform does

- **You (author):** create `skills/<name>/SKILL.md` (+ optional `reference/`),
  commit, push.
- **Platform (automatic):** scans the dir, builds the catalog, serves bodies via
  the `skill` tool. No manifest entry needed — skills are auto-discovered.

## Gotchas

- **Patch-existing takes effect this session; create-new does not.** The `##
  Available Skills` catalog (the `name — description` list in the prompt) is scanned
  **once per session** and frozen; the `skill` tool, by contrast, re-reads
  `<name>/SKILL.md` from disk on **every call**. So editing an *already-listed* skill
  takes effect the next time it's loaded this session, but a **brand-new** skill is
  absent from the frozen catalog — the employee can't reliably find it by name until
  the registry rebuilds (next session / new bundle commit SHA). Treat "save this as
  a skill" as a future-session affordance, not an in-session one.
- **Live at the next idle turn, not mid-turn** — bundle changes (incl. the catalog
  rebuild above) land when the agent rebuilds at the new commit on its next idle
  access, never mid-turn. No restart needed.
- **Missing `description` = silently skipped.** If a skill never loads, check the
  frontmatter first.
- A skill the employee "seems to use" without a `skill` tool call is **prompt
  knowledge, not a loaded skill** — confirm via the trace that the tool fired.
