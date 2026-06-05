# Guards & connectors — how the employee touches the outside world

## Why guards exist

An Ansur employee runs in a **deny-all sandbox**: a real Linux box with bash and
runtimes, but **no network egress** except a tiny static allowlist, and **no
credentials** — no API tokens, no OAuth keys, no cookies. It cannot
`curl https://gmail.googleapis.com/...`; the packet has nowhere to go.

A **guard** is the one door out for a given system. It's a **wire-level
transparent proxy**: the employee makes the *real* HTTP call it already knows how
to make (no special SDK, no MCP tool), and the guard mediates it. A guard does
three things:

1. **Holds the credential** — the employee never sees the token; the guard injects
   it on the way out.
2. **Enforces policy** — deterministic rules + optional LLM-judge checks run on
   every request; a violation returns a structured, parseable error.
3. **Audits** — every request (allowed or denied) is recorded outside the
   employee's reach.

So the employee can be fully compromised and still leak only the bundle/workspace
— never the upstream credential, and never more access than policy allows.

### The request path (in words)

One guard serves all of a tenant's agents for one system (per-`tenant × system`):

```
employee (sandbox)  →  real HTTP call to the system's host
  → identity sidecar (same pod, holds the mTLS key, presents a cryptographic identity)
  → Envoy guard (terminates mTLS, verifies the identity)
  → ext_authz policy: identity → which credential? decode body → run rules + judges
        ALLOW → inject the credential, forward          (Envoy → real upstream)
        DENY  → 403 with a parseable reason
        HOLD  → wait for human approve/deny
        503   → a dependency broke (distinct from "policy said no")
```

Identity is **cryptographic, not a header the employee sets** — the key lives in
the sidecar, and the credential the guard fetches is keyed off the *verified*
identity. Agent A can never select agent B's or another tenant's credential. The
guard **fails closed**: unknown identity, oversized/undecodable body, policy
reject, or a downstream outage all stop the request.

## Connectors — what you connect

A **connector** is "this tenant has connected system X; here's where its credential
lives." Connecting one is what makes a guard exist. Three classes:

| Class | Holds customer data? | Credential | Friction at `connector add` | Examples |
|---|---|---|---|---|
| **capability** | No — a commodity | **Platform-owned** shared key | **Zero** — instant | `web-search`, `browser` |
| **oauth-identity** | Yes — their account | Customer's own, via platform OAuth | Click "allow" in a browser | `gmail`, `slack` |
| **byok-identity** | Yes — their account | Customer's own, **pasted** | Paste an API key/token | `sap` |

```bash
ansur connector list --available        # the catalog: system + class + description
ansur connector list --available --json # adds credentialHint — and is the source of
                                         # truth for valid connectors.yaml `kind:` values
ansur connector add web-search           # capability — instant, no credential
ansur connector add gmail                # oauth — opens browser, you click allow, CLI polls
ansur connector add sap                  # byok — CLI reads the token from stdin
ansur connector probe gmail              # verify token + guard health end-to-end
ansur connector remove gmail             # disconnect → the guard is torn down
```

**The guard self-creates** when the connector becomes *connected*: immediately for
capability connectors, on the OAuth callback for `gmail`, on the credential paste
for `sap`. You don't provision anything.

> `github` is **not** in this catalog. It's a control-plane credential
> (`ansur github connect`) — see `references/initial-setup.md`. Never put it in
> `connectors.yaml`.

## `connectors.yaml` — what the employee may reach

In the bundle repo. The `kind:` list gates the sandbox's egress: it opens a door
to exactly each named guard-system and nothing else. Absent/empty ⇒ deny-all.

```yaml
connectors:
  - kind: gmail                  # REQUIRED, must be a known system, unique
  - kind: web-search
    config:                      # OPTIONAL — opaque per-kind startup config
      provider: exa
```

Validation: every `kind` must come from `ansur connector list --available`;
duplicate `kind` is rejected (one guard per kind).

**Connector `kind:` vs guards-repo `<system>/` dir** — these names differ. The
bundle and `connector add` use the **catalog** name; the guards repo and
`ansur guard pin` use the **wire guard-system** name:

| `connectors.yaml` / `connector add` | Guards repo dir / `guard pin` |
|---|---|
| `gmail` | `gmail/` |
| `web-search` | `web-search/` |
| `sap` | `sap-service-layer/` (writes) **and** `sap-hana/` (reads) |

Author policy under the **right-hand** path. A `guards/gmail/` dir does nothing
for an agent that only connects `sap`. **`sap` fans out to two guard-systems** —
one connector, two policy dirs — and the read path (`sap-hana/`) needs an
agent→role `groups:` mapping or it 403s. See **`references/sap.md`**.

> The bundle's `connectors.yaml` says *which* systems the employee may reach.
> Guard **policy** (what each call may do) lives in a **different repo** — see below.
> The legacy `extension: guards/<system>/` field in `connectors.yaml` still parses
> but the wire path **ignores** it — do not author bundle-local `guards/`.

## Guard policy lives in the per-tenant `<tenant>/guards` repo

Policy is **not** in the agent bundle. It's a separate repo, **one per tenant**,
that serves *all* of the tenant's agents (the wire guard is per-`tenant × system`).
Each wire guard git-fetches its `<system>/` subtree from `<tenant>/guards` at boot.

```
<tenant>/guards          ← one repo per tenant (not per agent)
├── gmail/
│   ├── rules.yaml        # deterministic checks
│   └── judges/
│       └── tone.md       # LLM-judge rubric, e.g. "reject non-professional tone"
└── sap-service-layer/
    └── rules.yaml
```

**Policy is data, never code** — you write rules + judge rubrics; the platform
ships the guard image. **No repo, or no `<system>/` dir = audit-only defaults:**
forward + audit every call, no checks. Add policy when the job needs a real
boundary (block competitor recipients, require a confirmation step before a write).

The repo owner is the GitHub account the App is installed on
(`<accountLogin>/guards`), **not** the internal customerId — the platform
derives it from the installation, so you never name it by hand.

Because one guard serves every agent in the tenant, scope a rule to specific
agents inside `rules.yaml` via the `agents:` / `groups:` rule scope.

### `mode:` — and what happens when no rule matches

The top of `rules.yaml` declares one `mode:` for the whole file. It governs the
**borderline** cases (an uncovered write, a held-for-approval verdict, a body the
guard couldn't decode). An explicit `reject_if` that fires is a **403 in both
`gated` and `enforced`** — mode only changes the borderline disposition:

| | `observe` (scaffold default) | `gated` | `enforced` |
|---|---|---|---|
| explicit `reject_if` true | forward + log "would block" | **403** | **403** |
| **write with no matching rule** | forward + log | **hold for human** approve/deny | **403** |
| needs-approval / undecodable body | forward + log | **hold for human** | **403** |
| read (`GET`/`HEAD`), or a covered call that passed | forward | forward | forward |

Two things the model hinges on:

- **Reads pass by default; writes fail closed.** A `POST`/`PUT`/`DELETE` that **no
  rule's `on:` matches** is treated as an *uncovered write* — denied under
  `enforced`, held under `gated`. So under enforcement, **every write you want to
  allow needs an explicit rule**; silence = deny. A `GET`/`HEAD` with no rule is
  forwarded (write a rule if you want to gate reads too).
- **`gated` puts a human in the loop; `enforced` is autonomous.** Pick `gated`
  when a person should approve borderline sends; `enforced` when the agent should
  be hard-blocked with no wait. `observe` blocks nothing — it's for harvesting what
  *would* be blocked before you commit to a boundary.
- **`approve_if` / judge `needs_approval` only hold under `gated`.** Under
  `enforced`, any `needs_approval` verdict (explicit `approve_if`, judge, or the
  gated-unknown fallback) becomes a terminal **403** — no hold, no Approve/Deny
  buttons. Under `observe`, explicit `approve_if` still **forwards** (audit-only).
  Human-in-the-loop email/SAP writes ⇒ **`mode: gated`**, not `enforced`.

Author rules under `mode: observe` first, watch the audit for `wouldBlock`
flags, then flip to `gated` / `enforced` once the rules cover the real traffic.

### Human-in-the-loop — `approve_if` in `<tenant>/guards` (load-bearing)

When the employee can **write** to an external system (Gmail send, SAP POST, …)
and a human must tap **Approve/Deny** before the side effect happens, you need
**two** artifacts — the guard policy *and* the bundle notify destination (below).
Setting only one leaves either silent forwards or approve buttons with nowhere to go.

**1. Guard policy** (`<tenant>/guards/<system>/rules.yaml`) — use `mode: gated`
and an explicit rule whose `approve_if` matches the writes you want held. For
Gmail outbound send (the common case), a minimal policy is:

```yaml
mode: gated

rules:
  - name: outbound-send-needs-approval
    on: { method: POST, path: /gmail/v1/users/me/messages/send }
    decode: rfc822
    approve_if: "true"
    reason: outbound send requires human approval
```

`approve_if: "true"` means every matching send queues for approval (the expression
language treats `"true"` as a literal). Narrow it later (`approve_if: count(payload.to) > 5`,
competitor-domain checks, etc.) — start broad when bootstrapping. Scope with
`agents:` / `groups:` when only some agents may send.

Without a matching rule under `gated`, an uncovered write still holds for approval
via the mode fallback — but an explicit `approve_if` rule is clearer, shows up in
audit with your `reason:`, and is what you want for “always approve sends.”

Validate before push: `ansur guards validate`, then `git commit && git push` in the
guards repo. See `packages/gmail-wire-adapter/examples/gmail/rules.yaml` for richer
examples (competitor blocks, judges).

**2. Bundle notify destination** — guard buttons are delivered to
`manifest.yaml` → `approvals.notify` in the **agent bundle** (not the guards repo).
That wiring is documented in `references/bundle.md`. Also required on the platform
side: **`ansur channel bind telegram …`** for that agent — proactive notify sends
through the **bound bot's token** to `approvals.notify.address` (the chat id can
be the operator's private chat, not the group the employee serves).

**3. Text fallback (no buttons).** The operator can always type `approve` / `deny`
(or `approve <apr_…>`) in the **same Telegram chat** as the employee when a turn
is parked — even without `approvals.notify`. Buttons are the upgrade; the text
commands are the stopgap.

Push **both** repos after edits: guards policy does not live in the bundle push.

### Judge rules (LLM checks) — the model is the platform's, not yours

A `judge:` rule sends the decoded payload to an LLM with your rubric
(`prompt_file`, e.g. `judges/tone.md`) and acts on the verdict
(`on_reject`/`on_accept`/`on_needs_approval`). Two non-obvious facts:

- **The judge model is platform config — and if it's missing, judge rules block
  everything.** The judge calls the platform's central LLM router (Brain) using a
  model the *operator* pins (`GUARD_JUDGE_MODEL`). If the platform hasn't wired a
  judge model, **every judge rule fails closed — it rejects every matching call**,
  regardless of how good your rubric is. So if a judge rule is blocking
  everything, suspect *platform judge config* before your prompt. You author the
  **rubric**; the platform owns the **model** (it runs on platform credits, not
  the tenant's).
- **You may request a per-rule `model:`, but it's bounded.** A judge rule can name
  a `model:`, but it's honored only if the operator allowlisted it; otherwise the
  platform's pinned model is used. Don't depend on a specific model unless you've
  confirmed it's allowed.

Write the rubric and the verdict actions; treat "is a judge model wired?" as an
operator question, not something the bundle or guards repo can fix.

### Provisioning the repo — `ansur guards init`

The guards repo doesn't exist until you create it. Until it does, every guard
runs **audit-only** (forwards + audits, blocks nothing) — so authoring policy is
gated on this one-time step:

```bash
ansur guards init        # create <accountLogin>/guards, seed an audit-only
                          # <system>/rules.yaml per connected system, then clone it
ansur guards clone       # later: re-clone the repo to edit (native git auth)
ansur guards validate    # offline check before every push (same loader as boot)
```

Run **`guards init` after `connector add`** — it seeds one `<system>/` dir per
*currently connected* wire guard-system. If the repo already exists,
`guards init` returns `repo_not_empty` — use `guards clone` and add missing
`<system>/` trees by hand (copy the shape from `guards init`'s scaffold or the
examples under `packages/*-wire-adapter/examples/`). Connecting a new system later
does **not** auto-create its policy dir.

`guards init` requires `ansur github connect` first (the owner comes from the
installation). The App **creates** the repo, so it auto-joins the installation —
no manual access grant. Each `<system>/rules.yaml` lands as `mode: observe` with
commented examples; edit, then — **ALWAYS run `ansur guards validate` before you
`git commit && git push`.** It runs the guard's *own* policy loader against every
`<system>/rules.yaml` (invalid `decode:`, malformed expression, missing judge
`prompt_file`) and exits non-zero on any error. This is not optional: a bad
`rules.yaml` makes the guard **fail closed (CrashLoopBackOff) at boot**, so the
push silently does NOT take effect — the guard keeps serving its last-good policy
and you get no signal. Validate first, fix what it reports, then push. The running
guard rolls onto the new policy on its next reconcile (or pin a commit with
`ansur guard pin`). If GitHub is connected to a **personal account** (not an org),
the App can't auto-create the repo — `guards init` returns an actionable error.
The agent runs `gh repo create <login>/guards --private` (empty), adds it to the
install the same way as a bundle repo (open
`https://github.com/settings/installations/<id>` in the customer's browser → add
the repo → Save → wait for confirmation; the `gh api PUT` grant 403s on a normal
token — see `bundle.md`), then re-runs `guards init`.

> **Legacy note:** an older v1 layout kept policy *inside* the agent bundle under
> `guards/<system>/`, pointed at by a `connectors.yaml` `extension:` field. That
> field still parses but the wire path ignores it — policy now lives in
> `<tenant>/guards`. Don't author bundle-local `guards/`.

## The real guards (status matters — don't over-promise)

| `kind:` | Class | Upstream | Status |
|---|---|---|---|
| `gmail` | oauth-identity | Gmail REST | **Live.** Injects the tenant's OAuth bearer. |
| `web-search` | capability | Exa | **Live.** Platform-owned key; strips any agent-supplied `x-api-key`. The customer never sees "exa." |
| `sap` | byok-identity | tenant's SAP Service Layer (writes, 50000) + HANA (reads, 30015) | **Live, both paths.** Writes via `sap-service-layer/`, reads via `sap-hana/`. Set connection config at connect time: `connector add sap --config '{"upstreamOrigin":…,"allowedCompanyDbs":[…],"defaultCompanyDb":…}'`. Two policy dirs + a required read role mapping + a HANA grant — the full recipe is **`references/sap.md`**. |
| `browser` | capability | Browserbase (CDP) | **Separate broker track — NOT wired to the wire-guard reconciler.** `kind: browser` parses but opens no wire egress today. |
| `slack` | oauth-identity | — | **Catalog only** — wire adapter pending. |

So `gmail`, `web-search`, and `sap` (reads + writes) are what actually works
through the wire path. Don't tell a customer `browser` or `slack` "just works"
like gmail.

## Production policy pins — `ansur guard pin`

A wire guard tracks `main` HEAD of the `<tenant>/guards` repo by default. For
staged release / rollback, freeze its policy at a commit:

```bash
ansur guard pins                      # show current pins
ansur guard pin sap <commit-sha>      # freeze sap's policy at that commit
ansur guard unpin sap                 # resume tracking main
```

## Author does vs. platform does

- **You (author):** declare the `kind:` list in the bundle's `connectors.yaml`;
  optionally write `<system>/` rules + judges in the **`<tenant>/guards` repo**
  (for human-in-the-loop writes: `mode: gated` + `approve_if` **and** bundle
  `manifest.yaml` `approvals.notify` — see above). The customer/operator runs
  `connector add` to connect the account.
- **Platform (automatic):** the guard image + decode logic, mTLS identity, the
  credential store + per-request brokering, OAuth refresh, the rule/judge engine,
  audit, guard creation/teardown, and the sandbox egress NetworkPolicy.

One line: **you declare *which systems* and *what policy*; the platform provisions
*the credential, the identity, the runtime, and the egress*.**
