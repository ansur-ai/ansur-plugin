# The agent bundle

**The bundle is the employee.** It's a single git repo — one per agent — that the
platform clones into the runtime sandbox at session start. It's **interpreted at
runtime** (bash runs directly, TypeScript via `tsx`, markdown/YAML read as-is) —
**no build step.** Editing the employee = editing files here and pushing.

## What it is — anatomy

| File / dir | Purpose | Required? |
|---|---|---|
| `manifest.yaml` | Identity + runtime config (model, reasoning, …). | **Yes** — loader throws if missing. |
| `prompt.md` | The one customer-authored system prompt → the employee's *identity*. Everything else in the prompt stack is platform-owned. **Most of your work goes here.** | **Yes** in practice. |
| `tools.yaml` | A `platform:` tool list. **Vestigial** — the platform injects the full tool set regardless; this can only add, never remove. | **Must exist & parse**; content does nothing. |
| `connectors.yaml` | Which of the tenant's connectors this agent exposes (`kind:` entries). Gates sandbox egress + triggers guard creation. Absent ⇒ no external systems. | Optional. → `references/guards.md` |
| `skills/<name>/SKILL.md` | On-demand playbooks the employee loads via the `skill` tool. | Optional. → `references/skills.md` |
| `memory/` | Mutable markdown the employee reads/writes; committed back across turns. | Optional. → `references/memory.md` |
| `hooks/{pre-tool,post-tool,stop}/` | Bash gates on the employee's agent loop. | Optional. → `references/hooks.md` |
| `oracles/*.ts` | Post-delivery success/failure scoring. | Optional (advanced). |
| `README.md` | Human-readable; not consumed at runtime. | Optional. |

> **Guard policy is NOT in the bundle.** It lives in a separate per-tenant repo,
> `<tenant>/guards` (one repo for all the tenant's agents). The bundle only
> *names* the systems via `connectors.yaml`. → `references/guards.md`

Decide where a constraint belongs by **when it acts**:
**hooks = pre-action (shape the loop) · guards = at-the-wire (mediate external calls) · oracles = post-delivery (score the outcome).**

## `manifest.yaml` — field reference

What the runtime loader actually reads (everything else is ignored):

```yaml
agent_id: bdr                 # REQUIRED — the agent's identity
model: claude-opus-4-7        # REQUIRED — supports model:reasoning, e.g. gpt-5.4-mini:high
reasoning: medium             # optional — none | low | medium | high
displayName: "BDR"            # optional — falls back to agent_id
provider: codex               # optional — derived from the model when omitted
role: sales                   # optional
maxIterations: 50             # optional — agent-loop cap (default 50)
heartbeat: { every: 1h }      # optional — scheduled self-prompt
approvals:                    # optional — where guard approve/deny prompts go
  notify: { kind: chat, channel: telegram, address: "<chat-id>" }
```

**Inert keys — present in the scaffold but ignored:** `bundleVersion`, `version`,
`description`. Do **not** rely on bumping `version` — it does nothing. The live
version is whatever commit is at `main` HEAD (or the pin). See *Versioning* below.

## Creating a bundle — `ansur bundle create <agent>`

Creates the GitHub repo, commits a starter scaffold, records the repo→agent
mapping, then clones it locally for you to edit.

```bash
ansur bundle create bdr            # org account: App creates the repo
ansur bundle create bdr --repo acme/bdr-bundle   # explicit repo name
```

The scaffold you start from:
- `manifest.yaml` (with `agent_id`, `model: claude-opus-4-7`, `reasoning: medium`)
- `prompt.md` (a persona stub — replace it)
- `tools.yaml` (`platform: [read, write]` + a "vestigial" comment)
- `connectors.yaml` (`connectors: []` + commented gmail/web-search examples)
- empty `skills/`, `memory/`, `hooks/pre-tool/`, `hooks/post-tool/` (via `.gitkeep`)
- `README.md`

`agent_id` must match `^[a-z0-9][a-z0-9-]*$`.

### Org vs personal-account repos

- **Organization** — the App creates the repo and GitHub auto-adds it to the
  "select repositories" install. Just run `bundle create`. Every later agent's
  repo auto-joins (one install covers all).
- **Personal account** — the App can't auto-create a repo there, so do the setup
  with the customer's own `gh`/git auth. The human's one GitHub step is adding the
  new repo to the install (everything else the agent does):
  1. `gh repo create <owner>/<name> --private` (empty, no README) — the agent runs this.
  2. Add the repo to the App installation. The API grant
     (`gh api --method PUT /user/installations/<id>/repositories/<repo_id>`) needs
     App-management permission a normal `gh` token usually lacks — it returns
     `403`, so don't rely on it. Reliable path: **open
     `https://github.com/settings/installations/<id>` in the customer's browser**
     (`<id>` from `gh api /user/installations`), **ask them to add `<owner>/<name>`
     under "Repository access" → Save, and wait for them to confirm** before
     continuing. (Not `ansur github connect` — that's a one-time link that expires;
     this is the live install-settings page.)
  3. `ansur bundle create <agent> --repo <owner>/<name>`.
  A `409 repo_precreate_required` means step 1 is missing.

Either way the repo must be **empty** — create refuses to clobber existing commits.

## Lifecycle — create → push → live

1. **Edit** the clone (`prompt.md`, `connectors.yaml`, `skills/`, …).
2. **Ship:** `git commit && git push` to `main`. There is no write API.
3. The Ansur **GitHub App webhook** fires; the platform records the commit and
   **advances the active pointer** for the agent to that commit SHA.
4. The running agent is marked **stale** and **hot-reloads at the next idle turn
   boundary** (~<30s in practice). A busy agent finishes its turn on the old
   bundle first — a push never kills a turn mid-flight.

To iterate: edit → push → `ansur trace <agent>` → repeat.

## Versioning — the active pointer

There is **no curated version axis** (it was retired). The model is:
- **Default:** the agent follows `main` HEAD — every push goes live.
- **Pin** (operator, for staged release / rollback): freeze at a specific commit.
  Live commit = `COALESCE(pinned, head)`.
- **Roll back:** pin an earlier commit, or `git revert` + push.

`ansur bundle list` shows each agent's active commit (HEAD or pinned) + repo URL.

## Cloning later — `ansur bundle clone <agent> [dir]`

Resolves the repo URL and shells to native `git clone` (your machine's own git
auth). Use it to re-open a bundle on a fresh machine.

## Author does vs. platform does

- **You (author):** write `prompt.md`; declare `connectors.yaml`; optionally add
  `skills/`, `hooks/`, `memory/` seeds; set `manifest.yaml` identity; `git push`.
  (Guard *policy* goes in the separate `<tenant>/guards` repo — see
  `references/guards.md`.)
- **Platform (automatic):** provisions the repo + scaffold; injects the full tool
  set; supplies every prompt layer except identity; on push advances the pointer +
  hot-reloads; generates sandbox egress from `connectors.yaml`; reconciles guards;
  routes channels (channels are tenant-level, never bundle-defined).
