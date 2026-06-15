# SAP — connecting the SAP Business One guard(s)

SAP is the one connector that fans out to **two** wire guard-systems from a single
`connector add sap`:

| Guard-system (guards-repo dir) | Path | Port | What it does |
|---|---|---|---|
| `sap-service-layer/` | **writes** | 50000 | B1 Service Layer OData v2 (create/update production orders, …) |
| `sap-hana/` | **reads** | 30015 | direct HANA SQL (`SELECT …`) |

One `sap` connector and one `connectors.yaml` `kind: sap`, but **two policy dirs**
and two credential shapes. Both reads and writes pick the target company DB (= HANA
schema) **per request** via an `x-sap-company-db` header, gated by the **same**
`allowedCompanyDbs` allow-list on the connector — they just authenticate to
different upstreams:

- **Reads** hit HANA as a SQL reader user (keyed by role via `sap-hana.users`).
- **Writes** log into the Service Layer as a B1 user; the B1 session binds to the
  selected company DB at login, so the guard validates the selector against
  `allowedCompanyDbs` **before** minting the session.

## The three keys + the shared schema config

| Key | Where | read/write | Meaning |
|---|---|---|---|
| `sap.username` / `sap.password` | secret | **write** | Service Layer (B1) login credentials |
| `sap-hana.users` | secret | **read** | HANA reader users keyed by role: `{"<role>":{"user","password"}}` |
| `allowedCompanyDbs` / `defaultCompanyDb` | connector **config** | **read + write** | the company DBs an agent may target (allow-list) + the default when no `x-sap-company-db` is set — governs BOTH paths |

There is **no `sap.companyDb` secret** — the writable/readable schema set is the
connector's `allowedCompanyDbs` config, one source of truth for reads and writes. A
company DB **is** a HANA schema (e.g. `DENEME_URETIM_TEST2`) — same name on both
sides. To allow N schemas (reads or writes), list N in `allowedCompanyDbs`; the
guard 403s anything off the list.

## Connect it (CLI-native)

```bash
# 1. Add the connector WITH its connection config (the --config flag). The
#    allow-list governs BOTH reads and writes; list every schema agents may touch.
ansur connector add sap --config '{
  "upstreamOrigin": "https://<sap-host>:50000",
  "allowedCompanyDbs": ["FACTORY_A", "FACTORY_B", "FACTORY_C"],
  "defaultCompanyDb": "FACTORY_A"
}'
#   add then prompts for the credential — paste the Service Layer (B1) WRITE login
#   as JSON: {"username":"<b1-user>","password":"<b1-pass>"}. The connect
#   materializes it into the sap.username / sap.password secrets the write guard
#   reads, and REFUSES to connect (400) without both — there is no "connected but
#   uncredentialed" state, and the write guard crashloops if it boots without them.
#   No sap.companyDb — the schema is per-request (x-sap-company-db), allow-listed
#   by the config above. (Equivalent manual seed, instead of the paste:
#     printf '%s\n' '<b1-user>' | ansur secret set sap.username
#     printf '%s\n' '<b1-pass>' | ansur secret set sap.password)

# 2. Secret — READ login (HANA reader, keyed by role):
printf '%s\n' '{"super_read":{"user":"SAP_GUARD_SUPER_READER","password":"<pw>"}}' \
  | ansur secret set sap-hana.users

# 3. Guards repo (scaffolds BOTH sap-service-layer/ and sap-hana/):
ansur guards init
#   → EDIT sap-hana/rules.yaml — under groups:, map your agent to the read role:
#       groups:
#         super_read: [<agent-id>]     # role name MUST match a sap-hana.users key
#   → sap-service-layer/rules.yaml: leave `mode: observe` (writes pass) or add rules
ansur guards validate && (cd <guards-clone> && git add -A && git commit -m sap && git push)

# 4. Bundle: declare the connector in connectors.yaml → `- kind: sap` → commit + push.
```

`secret set` reads the value from **stdin** — pipe it WITH a trailing newline
(`printf '%s\n'`), never as an argv.

## The read role mapping is REQUIRED — or every read 403s

`sap-hana` reads pick the HANA user by **role**: the `groups:` block in
`sap-hana/rules.yaml` maps an agent id to a group, the group **name** is injected as
`x-guard-role`, and the read guard looks that name up in the `sap-hana.users`
secret. An agent in **no group → empty role → every read returns 403** with no
obvious reason. The role name in `groups:` and the key in `sap-hana.users` MUST
match (e.g. both `super_read`). The `guards init` scaffold ships an active (empty)
`groups:` block precisely to force this.

## How the employee calls SAP at runtime (for the bundle author)

When `kind: sap` is connected the sandbox injects two env vars; the employee makes
plain HTTP calls — the guard injects auth + forwards, so there is **no login step**
and the employee never sees a credential:

```bash
# READ (HANA SQL): POST the SQL; pick the schema with x-sap-company-db.
curl -X POST "$SAP_HANA_QUERY_URL" -H 'content-type: application/json' \
  -H 'x-sap-company-db: DENEME_URETIM_TEST2' \
  -d '{"sql":"SELECT TOP 5 \"ItemCode\" FROM OITM"}'
#   → { ok, columns, rows, rowCount }    (a disallowed schema → 403 schema_not_allowed)

# WRITE (Service Layer): POST to the SL path; pick the schema with x-sap-company-db
# (omit to use defaultCompanyDb). The guard validates it against allowedCompanyDbs,
# then logs into that company DB.
curl -X POST "$SAP_BASE_URL/b1s/v2/ProductionOrders" -H 'content-type: application/json' \
  -H 'x-sap-company-db: FACTORY_B' \
  -d '{"ItemNo":"…","PlannedQuantity":1,"Warehouse":"…","ProductionOrderType":"bopotStandard","ProductionOrderOriginEntry":<salesOrderDocEntry>,"DueDate":"2026-06-30T00:00:00"}'
#   → 201 with the created order   (a disallowed schema → 403 before any login)
```

The bundle prompt should list the writable schemas (so the agent addresses the
right one) — but the prompt is only guidance; `allowedCompanyDbs` is the enforcement.

## Prerequisites you can't do from the CLI (operator / SAP side)

Infra, not bundle/connector config — flag these to the operator if a SAP call
fails *before* policy:

1. **IPsec tunnel** from the cluster to the on-prem SAP host (Service Layer 50000 +
   HANA 30015). Without it the guard's upstream is unreachable.
2. **A HANA reader user GRANTed on the target schema** (e.g. `GRANT SELECT ON
   SCHEMA "DENEME_URETIM_TEST2" TO SAP_GUARD_SUPER_READER`). The user in
   `sap-hana.users` must have read rights on every company DB in `allowedCompanyDbs`,
   or reads return insufficient-privilege.
3. **A B1 Service Layer user** that can log into every company DB in
   `allowedCompanyDbs` (writes).

## Gotchas

- **Reads need the `groups:` mapping** (above) — the #1 silent failure.
- **Secret typos fail closed.** A wrong `sap.username`/`sap.password` is not a clear
  error — the guard's login fails and Envoy returns a bare **403** on the write.
  Re-check the secret values before suspecting policy.
- **Writes (like reads) are scoped by `allowedCompanyDbs`**, not a single secret —
  the agent picks per write via `x-sap-company-db`, the guard 403s anything off the
  list before logging in. Allow N schemas by listing N. A write with no header and
  no `defaultCompanyDb` fails closed.
- **Two policy dirs, one connector.** `sap` → both `sap-service-layer/` (writes) and
  `sap-hana/` (reads). Author policy under both as the job needs.
- **`connector add sap --config` is the config path.** Older notes saying
  `upstreamOrigin` must be set via API/DB are obsolete.
