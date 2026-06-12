# Sovos — connecting the e-fatura (e-invoice) guard

Sovos / FİT Bulut is the Turkish e-Invoice cloud. One `connector add sovos` →
**one** wire guard-system (`sovos`), public egress (`efaturaws.fitbulut.com`). The
employee calls the Sovos SOAP WS directly over HTTPS; the guard intercepts, reads
the `VKN_TCKN` out of the request body, and injects **that company's** HTTP Basic
credential. The employee never sees a credential.

## The model: one tenant, many companies, one secret

A tenant often invoices as **several legal entities** (e.g. ANSA Ambalaj +
Agroplast), each with its own Sovos WS account. Sovos is **`companies`-scoped**:
the request's `VKN_TCKN` (tax number, already in the invoice payload) selects which
company — and therefore which credential — to use. So a Sovos connect carries **two
coupled things**:

| What | Where | Secret? | Shape |
|---|---|---|---|
| `companies` — VKN → company name | connector **config** (`--config`) | No | `{"<VKN_TCKN>":"<company>"}` |
| per-company Basic creds | one connector **secret** (`connector/sovos/<instance>`) | Yes | `{"<company>":{"username","password"}}` |

The **company names must match** across the two — a VKN maps to a name in
`companies`, and that name keys into the credential map. A VKN not in `companies`,
or a company missing from the credential map, **fails closed** (no default
credential).

> **Both halves are required at connect time.** The `companies` map is what the
> reconciler stamps as the guard's `SOVOS_COMPANIES` env. **Without it the guard
> fails closed at boot (CrashLoopBackOff)** — the connect now rejects a Sovos
> credential POST that has no `companies` map, so you find out at `connector add`,
> not when the agent's first e-invoice silently can't go out.

## Connect it (CLI-native)

```bash
# Add the connector WITH the companies map (the --config flag), then paste the
# per-company credential map at the prompt.
ansur connector add sovos --config '{
  "companies": {
    "<ANSA_VKN>": "ANSA",
    "<AGROPLAST_VKN>": "AGROPLAST"
  }
}'
#   prompt: paste the credential as ONE JSON object, company → {username, password}:
#   {"ANSA":{"username":"<u>","password":"<p>"},"AGROPLAST":{"username":"<u>","password":"<p>"}}
```

That single credential POST stores the secret **and** the config, and fires the
wire reconcile — the guard comes up with `SOVOS_COMPANIES` stamped.

**Test vs live cloud.** Default upstream is live (`efaturaws.fitbulut.com`). A
tenant on the test WS adds `"upstreamOrigin":"https://efaturawstest.fitbulut.com"`
to `--config`.

**One connector, both companies — not one instance per company.** Put every entity
in the *same* connector's `companies` map + credential secret. Don't create a
second `--instance` per company: both instances map to the single `sovos` guard and
collide. (If a box already has stray instances, `connector remove sovos --instance
<x>` them.)

## Changing the map later

`connector add` on an existing row returns **409 "already exists"**, and `--config`
only rides a *fresh* credential POST. So to change `companies` (add/rename an
entity), **remove then re-add**:

```bash
ansur connector remove sovos
ansur connector add sovos --config '{"companies":{ … updated … }}'   # re-paste creds
```

A pure credential rotation that keeps the same companies can omit `--config` — the
stored map still satisfies the connect check.

## How the employee calls Sovos at runtime (for the bundle author)

The agent calls the Sovos WS host **directly** (transparent HTTPS — no injected URL
env, no login step). It sends the SOAP request with the target entity's `VKN_TCKN`
in the body; the guard decodes it, picks that company, injects `Authorization:
Basic …`, and forwards. The bundle's e-fatura skill just builds the SOAP envelope
and POSTs to the configured WS endpoint.

## Prerequisites you can't do from the CLI (operator / Sovos side)

- **Sovos allowlists the guard's SOURCE IP per WS account.** Each company's Sovos WS
  account must allow the cluster's egress IP — a **Sovos-side ticket**, not a
  cluster egress class. A correct credential still 401/403s from Sovos until the
  source IP is allowed.

## Gotchas

- **No `companies` map ⇒ guard CrashLoopBackOff** (`SOVOS_COMPANIES is required`).
  The connect check now blocks this up front; if you see a crashlooping
  `*-sovos-authz`, its connector config lost the map — re-add with `--config`.
- **Company names must match** between the `companies` config and the credential
  map keys, exactly. A mismatch fails that company closed.
- **The VKN is the request's, not yours to invent.** The map keys must be the real
  tax numbers that appear in the SAP/invoice payloads — that's what the guard reads
  to route.
- **`connected` in `connector list` ≠ provisioned.** It only means the row exists;
  confirm the guard pod is `Running`, not crashlooping, after a connect.
