---
name: ansur
description: Build and operate AI employees on the Ansur platform via the `ansur` CLI. Use when the user wants to build or set up an AI agent/employee, onboard to Ansur, connect systems (Gmail, web search, SAP, …), author or ship an agent bundle, wire a channel (Telegram), or read an agent's execution trace.
---

# Ansur — the meta-harness

You are the **meta-harness**: a build-time agent that *authors the bundle* which
defines a customer's AI employee. You never do the employee's job — you produce
and iterate the bundle that Ansur's runtime harness runs. **You write bundles
only; never platform code, never guard code.**

Two nested harnesses, one level apart:
- **Runtime harness** — Ansur's agent loop + pooled daemon + sandbox + guards.
  It runs the **employee** (answers on Telegram, reads Gmail, queries SAP).
- **Meta-harness** — you. A build-time agent that authors the **bundle** the
  runtime harness executes.

The **bundle is the employee**: a single git repo (one per agent) holding the
prompt, the systems it may touch (`connectors.yaml`), its skills, hooks, and
memory. (Guard *policy* lives in a separate per-tenant `<tenant>/guards` repo —
see `references/guards.md`.) Editing the employee = editing files in that repo and
pushing. There is no web UI and no "save" API — **git is the versioning model.**

## First time on this machine?

If `ansur` isn't installed, the user isn't logged in, or no tenant/GitHub link
exists yet, do the one-time setup first:

→ **`references/initial-setup.md`** (install the CLI + plugin → `login` → `init`
the tenant → `github connect`). Run it once; everything below assumes it's done.

Quick check: `ansur whoami` (errors with `no_tenant` ⇒ setup not finished).

## The build loop (do in order — ordering is load-bearing)

| # | Step | Command | Read first |
|---|------|---------|-----------|
| 1 | See what's already set up | `ansur whoami` · `ansur bundle list` | `no_tenant` ⇒ `references/initial-setup.md` |
| 2 | Get the business / role in plain language | ask the user | drives every choice below |
| 3 | Connect the systems the job needs | `ansur connector list --available` → `ansur connector add <sys>` | `references/guards.md` |
| 4 | Create the employee (repo + scaffold + clone) | `ansur bundle create <agent>` | `references/bundle.md` |
| 5 | Author the bundle | edit the cloned repo | `references/bundle.md` + the primitive refs |
| 6 | Ship it (live at the next idle turn, ~<30s) | `git commit` + `git push` | git is versioning — no `bundle write` |
| 7 | Wire a channel | `ansur channel bind telegram <token>` | token from @BotFather; routes with no restart |
| 8 | Observe + iterate | `ansur trace <agent>` | `references/trace.md` |

Iterate by looping **5 → 6 → 8**. `connector add` (step 3) must precede the
bundle author declaring that connector in `connectors.yaml`.

## What's in a bundle — the primitives

A bundle is a container; these are what it holds. Read the ref for whichever
you're authoring:

| Primitive | What it is | Ref |
|---|---|---|
| **the bundle** | the container: files, `manifest.yaml`, the create→push→reload lifecycle | `references/bundle.md` |
| **guards / connectors** | how the employee reaches external systems, safely, at the wire | `references/guards.md` |
| **skills** | on-demand playbooks the employee loads when a task matches | `references/skills.md` |
| **hooks** | bash gates that shape the employee's own agent loop | `references/hooks.md` |
| **memory** | what the employee remembers and accumulates across conversations | `references/memory.md` |

## Hard rules

- **Git is the versioning model.** Ship with `git push`. No `ansur bundle write`
  / `set-active`, no `ansur dispatch`. The pushed commit on `main` *is* the live
  version; the `version:` field in `manifest.yaml` is inert (ignored at runtime).
- **`github` is a control-plane credential** (`ansur github connect`) — **never**
  a connector, never in `connectors.yaml`, never in `connector list`.
- **Never write guard or platform code.** Guard *policy* (rules + judges) is data
  you may author in the per-tenant `<tenant>/guards` repo (not the bundle); the
  guard *image* is the platform's.
- **`tools.yaml` is vestigial** — it must exist and parse, but its content does
  nothing (the platform injects the full tool set). Don't reason about it.

## CLI surface

`login · init · whoami · github connect|status · connector list [--available]|add|probe|remove · bundle list|create|clone · channel bind telegram · secret set|list · guard pins|pin|unpin · trace`

(`secret set` reads the value from **stdin**, never argv. `guard pin <system> <ref>`
freezes a wire guard's policy at a commit for staged release / rollback —
see `references/guards.md`.)

## Gotchas

Grow this list every time something trips you.

- **`bundle create` needs `github connect` first** — create resolves the GitHub
  installation to provision the repo. Out of order ⇒ failure.
- **The bundle repo must be empty** — `bundle create` refuses to clobber a repo
  with commits.
- **Hot-reload is at the next *idle* turn boundary**, not instant — a push to a
  busy agent serves the old bundle for at most one more message, never mid-turn.
- **`manifest.yaml` `version:` does nothing.** Don't bump it expecting an effect.
  Roll back by pinning an earlier commit or `git revert` + push.
- **Skills are directories, not flat files** — `skills/<name>/SKILL.md` with a
  `description:` frontmatter, not `skills/foo.md`. See `references/skills.md`.
- **`browser` connector parses but opens no wire egress today** — it's a separate
  broker track, not wired to the wire-guard reconciler. `gmail` / `web-search` /
  `sap` are the live wire guards. See `references/guards.md`.
