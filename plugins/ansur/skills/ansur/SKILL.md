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
see `guards/guards.md`.) Editing the employee = editing files in that repo and
pushing. There is no web UI and no "save" API — **git is the versioning model.**

## First time on this machine?

If `ansur` isn't installed, the user isn't logged in, or no tenant/GitHub link
exists yet, do the one-time setup first:

→ **`setup/initial-setup.md`** (install the CLI + plugin → `login` → `init`
the tenant → `github connect`). Run it once; everything below assumes it's done.

Quick check: `ansur whoami` (errors with `no_tenant` ⇒ setup not finished).

## The build loop (do in order — ordering is load-bearing)

| # | Step | Command | Read first |
|---|------|---------|-----------|
| 1 | See what's already set up | `ansur whoami` · `ansur bundle list` | `no_tenant` ⇒ `setup/initial-setup.md` |
| 2 | Get the business / role in plain language | ask the user | drives every choice below |
| 3 | Connect the systems the job needs | `ansur connector list --available` → `ansur connector add <sys>` | `guards/guards.md` · **SAP: `guards/sap.md`** · **Sovos: `guards/sovos.md`** |
| 3b | Provision the guards policy repo (once) | `ansur guards init` (after step 3) | `guards/guards.md` — seeds `<system>/` per *connected* connector |
| 4 | Create the employee (repo + scaffold + clone) | `ansur bundle create <agent>` | `bundle/bundle.md` |
| 5 | Author the bundle | edit the cloned repo | `bundle/bundle.md` + the primitive refs |
| 5b | Author guard policy from the job's rules | edit the guards repo clone | **`guards/rules.md` (the grammar)** + `guards/guards.md` (modes); approvals also need `manifest.yaml` (`bundle/bundle.md`) |
| 6 | Ship **both** repos | `git commit` + `git push` in bundle **and** guards clones | manifest + policy are separate pushes |
| 7 | Wire a channel | `ansur channel bind telegram <token> [--agent <name>]` | token from @BotFather; **required** for approval buttons (see gotchas) |
| 8 | Observe + iterate | `ansur trace <agent>` | `operate/trace.md` |

Iterate by looping **5 → 6 → 8**. `connector add` (step 3) must precede
`guards init` (3b) and the bundle declaring that connector in `connectors.yaml`.
If you connect a *new* system after `guards init`, add `<system>/rules.yaml`
manually in the guards repo (init does not re-run).

## What's in a bundle — the primitives

A bundle is a container; these are what it holds. Read the ref for whichever
you're authoring:

| Primitive | What it is | Ref |
|---|---|---|
| **the bundle** | the container: files, `manifest.yaml`, the create→push→reload lifecycle | `bundle/bundle.md` |
| **guards / connectors** | how the employee reaches external systems, safely, at the wire | `guards/guards.md` |
| **guard rules** | the rule LANGUAGE — turn the job's "never X / hold Y for a human" into `rules.yaml` | `guards/rules.md` |
| **skills** | on-demand playbooks the employee loads when a task matches | `bundle/skills.md` |
| **hooks** | bash gates that shape the employee's own agent loop | `bundle/hooks.md` |
| **memory** | what the employee remembers and accumulates across conversations | `bundle/memory.md` |

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

`login · init · whoami · github connect|status · connector list [--available]|add|probe|remove [--instance <name>] [--config '<json>'] · guards init|clone|validate [dir] · guard pins|pin|unpin|status|rollback · bundle list|create|clone [--repo owner/name] · channel bind telegram <token> [--agent <name>] · secret set|list · trace <agent> [--since …]`

(`secret set` reads the value from **stdin**, never argv. `guards validate` runs the
guard's own policy loader **and the policy's behavioral `examples:`** offline (a
rule that diverges from its declared intent fails) — run before every guards-repo
push; the **control-plane publish gate** runs the same check at the daemon before
advancing the live ref, so a bad push never crashloops the guard. `guard pin
<system> <ref>` freezes a wire guard's policy at a commit; `guard status <system>`
shows intent → enforced → published history; `guard rollback <system>` reverts to a
prior published SHA — see `guards/guards.md`. Global flags: `--json`, `--endpoint`.)

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
  `description:` frontmatter, not `skills/foo.md`. See `bundle/skills.md`.
- **`browser` connector parses but opens no wire egress today** — it's a separate
  broker track, not wired to the wire-guard reconciler. `gmail` / `web-search` /
  `sap` are the live wire guards. See `guards/guards.md`.
- **Gated writes need two files, not one.** `<tenant>/guards/<system>/rules.yaml`
  with `mode: gated` + `approve_if` (e.g. Gmail send → `approve_if: "true"`) **and**
  the bundle's `manifest.yaml` `approvals.notify` (Telegram `channel` + `address`).
  Either alone ⇒ no buttoned approve flow. `bundle create` does not scaffold either;
  add them when the role can send email or other guarded writes.
- **Human-in-the-loop requires `mode: gated`, not `enforced`.** Under `enforced`,
  a `needs_approval` verdict (including `approve_if`) is a terminal **403** — no
  hold, no buttons. Only `gated` waits for a human.
- **Connector `kind:` ≠ guards-repo directory.** `connectors.yaml` uses catalog
  names (`sap`); policy dirs use wire guard-system names. `sap` alone fans out to
  **two** dirs — `sap-service-layer/` (writes) **and** `sap-hana/` (reads). See the
  mapping table in `guards/guards.md` and the full recipe in `guards/sap.md`.
- **Approval buttons need `channel bind`.** Proactive notify sends via the bound
  bot's token to `approvals.notify.address` — bind first; the operator must have
  `/start`ed that bot in Telegram before DMs/buttons can arrive.
- **SAP is two guard-systems behind one connector** (`sap-service-layer/` writes +
  `sap-hana/` reads), with read/write-specific secrets and a **required** read role
  mapping (reads 403 without it). Connect with
  `ansur connector add sap --config '{"upstreamOrigin":…,"allowedCompanyDbs":[…],"defaultCompanyDb":…}'`.
  Don't wing it — follow **`guards/sap.md`**.
- **Sovos (e-invoice) needs a `companies` map at connect time** — it routes each
  request to a per-company credential by the payload's `VKN_TCKN`. Connect with
  `ansur connector add sovos --config '{"companies":{"<VKN>":"<company>"}}'` + a
  paste of `{company:{username,password}}`. **Omit the map and the guard fails
  closed at boot.** Follow **`guards/sovos.md`**.
