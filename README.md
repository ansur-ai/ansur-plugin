# Ansur — Claude Code plugin

> **Status: published.** Distributed via the public marketplace repo
> [`ansur-ai/ansur-plugin`](https://github.com/ansur-ai/ansur-plugin) —
> `/plugin marketplace add ansur-ai/ansur-plugin` then `/plugin install ansur@ansur`.
> This monorepo directory is the authoring source; the public repo is the
> published mirror (`plugins/ansur/` there). Keep the two in sync when editing.

Turns Claude Code into the **meta-harness** — the build-time agent a customer pastes
a prompt into, which drives the `ansur` CLI (`@ansur-ai/cli`) to build their AI
employee on the Ansur platform.

## The meta-harness (why this exists)

Two nested harnesses:
- **Runtime harness** — Ansur's own agent loop (TAOR) + pooled daemon + sandbox +
  guards. It runs the **employee**.
- **Meta-harness** — the **build-time** agent (Claude Code + this plugin). It
  **authors the bundle** the runtime harness executes. "Meta" = one level up: it
  doesn't do the employee's job, it produces the artifact (the bundle) that defines
  an agent that does.

Its only job is **writing + iterating bundles — never platform code**. A plugin (vs.
MCP) ships *behavior* (a skill + commands) that shapes *how* the agent
reasons, so the customer's agent behaves like Ansur's server-side meta-harness would
— and the customer pays that inference. The CLI is the small action surface.

Background (monorepo paths — for plugin authors editing this source, not
shipped to customers): `docs/design-docs/platform-architecture.md` (the
meta-harness loop), `docs/design-docs/platform-interface.md` (plugin + CLI, why
no web UI), `docs/design-docs/core-beliefs.md` (voice: real files/tools, context
discipline, Oracle thinking).

## Structure — one skill, a spine + on-demand depth by domain

The plugin ships a **single skill** (`ansur`) whose `SKILL.md` is a **spine /
lookup table** of the flow. Depth lives in domain directories the spine points
to and the agent reads on demand — not in separate registered skills.

```
plugins/claude-code/
  .claude-plugin/plugin.json   manifest
  commands/ansur-onboard.md    single slash entry point → loads the skill
  skills/ansur/
    SKILL.md                   the spine: mental model, build loop, hard rules (always loaded)
    setup/
      initial-setup.md         ONE-TIME: install + login + init + github connect
    bundle/                    the employee artifact
      bundle.md                the container: anatomy, manifest fields, create→push→reload lifecycle
      skills.md                on-demand playbooks the employee loads
      hooks.md                 bash gates on the employee's agent loop
      memory.md                what the employee accumulates across conversations
    guards/                    the safety boundary
      guards.md                connectors + wire guards: how the employee reaches external systems
      rules.md                 the rule LANGUAGE: expression grammar, when:/preflight:, authoring discipline
      sap.md                   the SAP connector recipe (two guard-systems, secrets, read role)
    operate/
      trace.md                 trace record shape + jq recipes
```

## Encoded "don'ts"

- No `ansur dispatch`; no `ansur bundle write`/`set-active` (git is the versioning model).
- `github` is **not** a connector — `ansur github connect` is a control-plane credential.
- Never write guard code (the platform ships the guard); `tools.yaml` is vestigial.

## Distribution

Published via a Claude Code plugin marketplace; the Track F3 install prompt installs
this plugin + `@ansur-ai/cli` and runs `ansur login`. Codex CLI / Cursor variants
reuse the same skill + CLI; only packaging differs.
