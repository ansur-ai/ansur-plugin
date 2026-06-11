# The rule language — authoring `rules.yaml` from a job spec

This is the grammar reference for guard policy. `guards.md` explains the *system*
(modes, the publish gate, human-in-the-loop); this file teaches the *language* so
you can turn a customer's job rules ("never book the same invoice twice", "hold
foreign-currency documents for a human") into deterministic checks. You author
rules from the job spec — there is no canned policy to copy; the policy IS the
customer's business rules, written in this grammar.

**The language is the same for every guard-system** (gmail, web-search, SAP, …) —
what differs per system is the `payload` shape its `decode:` produces and the
upstream paths you match/fetch. System-specific notes (SAP's cross-upstream SQL
preflight, `$batch`) are at the bottom, clearly marked.

## A rule, field by field

```yaml
mode: gated                  # whole-file posture — see guards.md

groups:                      # optional: name agent sets once, reference in rules
  writers: [satin-alma-bot]

rules:
  - name: my-rule            # unique; shows up in audit + 403 bodies as `check`
    agents: [writers]        # optional scope: agent ids and/or group names; absent = every agent
    on: { method: POST, path: /Things }   # which requests this rule governs
    decode: json             # how the body becomes `payload` (json | rfc822 | form | soap | none)
    when: "…"                # optional applicability gate (see below)
    preflight: { … }         # optional live-state fetch (see below)
    reject_if: "…"           # expression; true ⇒ 403
    approve_if: "…"          # expression; true ⇒ hold for human (gated) / 403 (enforced)
    reason: "…"              # human + agent readable; rides the 403 and the audit record
    on_preflight_error: reject   # or needs_approval — verdict when the preflight FETCH fails
    present: "…"             # optional: the operator's approval-card body (see below)
```

A rule needs at least one of `reject_if` / `approve_if` (or `judge:` — see
guards.md). **Engine precedence:** any matched rule's `reject_if` firing wins over
every `approve_if`; the first firing `approve_if` decides otherwise. Rules are
independent — write several small rules, not one mega-expression.

### `on:` matching — canonical paths

`on.path` is a **glob where `*` matches within one path segment** (`/a/*/c` does
not match `/a/b/d/c`), against the **canonical** path: the system adapter strips
its API prefix and any `(key)` — e.g. SAP's `/b1s/v2/PurchaseInvoices(123)`
matches `on: { path: /PurchaseInvoices }`. Non-canonical paths (e.g. a legacy
`/v1/...`) stay raw, so you can write a rule that matches and blocks them.
`on.method` is case-insensitive exact; omit it to match every method.

## The evaluation context — what expressions can read

| Root | What it is |
|---|---|
| `payload` | the decoded request body, per `decode:`. Shape is **system-specific** — for SAP it is `{ entity, operation, key, fields }` where `fields` is the parsed JSON body, so a body field reads as `payload.fields.DocTotal`. For gmail (`rfc822`) you get parsed mail fields (`payload.to`, `payload.subject`, `payload.body`). |
| `request` | `{ method, path, headers }` — `path` is the canonical path |
| `context` | `{ tenant, agent, system }` — the **cryptographically verified** caller; `context.agent == "x"` is an authorization check, not a hint |
| `preflight` | `{ inspect: <result of the preflight's inspect expression> }` — only on rules with `preflight:` |

Discover the payload shape empirically: run one real call under `mode: observe`
and read the audit record (`ansur trace`), or ask the operator for a sample
document. **Never guess field semantics** — see "Authoring discipline" below.

## `present:` — the operator's approval card

When a rule **holds** a request for a human (a matched `approve_if`, or a judge
`needs_approval`), the operator gets an Approve/Deny card. By default that card is
the generic `system / rule / reason / <scrubbed summary>` layout. `present:`
replaces the **body** with a plain-language card you author — in the operator's
own language — rendered from the **same evaluation context** the rule reads:

```yaml
  - name: ap-invoice-needs-approval
    on: { method: POST, path: /PurchaseInvoices }
    decode: json
    approve_if: "true"
    reason: "Satıcı faturası — insan onayı bekliyor"
    present: |
      🔐 Onay gerekli — Satıcı Faturası
      Tedarikçi: {{ payload.fields.CardCode }}
      Şirket: {{ payload.fields.U_Sirket }} · Fatura No: {{ payload.fields.NumAtCard }}
      Tutar: {{ payload.fields.DocTotal | default('hesaplanacak') }}
      Kalem: {{ payload.fields.DocumentLines | length }} satır
```

So instead of `{"entity":"PurchaseInvoices","bodyBytes":812}`, the operator reads
`Tedarikçi: …, Tutar: … TRY, Şirket: ANSA` and can actually decide.

**Syntax.** Literal text interleaved with `{{ path | filter | filter:arg }}`:

- `{{ payload.fields.CardCode }}` — a dotted path into the **same** `payload` /
  `request` / `context` roots the table above lists (`{{ context.agent }}`,
  `{{ request.path }}`). No expression logic — just a path and optional filters.
- Filters: `{{ x | default('—') }}` (fallback when the field is missing/empty —
  use it for server-computed fields like SAP's `DocTotal` that the POST body
  often omits), `{{ x | truncate:120 }}` (cap a long field), `{{ xs | length }}`
  (array/string length).

**Two guarantees, by design:**

- **Compile-time strict.** A malformed template (unbalanced `{{ }}`, an unknown
  filter, a bad arg) **fails `ansur guards validate`** and the policy load — same
  loud gate as a bad `approve_if`. Last-good policy stays live.
- **Render-time total.** At the hold it never throws: a missing path renders its
  `| default(…)` or empty, the rest of the card intact. `present:` is
  **display-only** — it never affects the verdict, so a render fault degrades to
  the legacy card and can never reject a write.

**Notes.**
- Renders **only** on a hold (`needs_approval`). A rule that accepts or rejects
  never shows it.
- `payload.fields.*` need the rule's `decode:` (a SAP write rule already declares
  `decode: json`). Don't add `decode:` just for the card — `request.*` /
  `context.*` resolve regardless, and decode changes verdict behavior (an
  unparseable body fails closed *before* the card renders).
- It is **not** PII-scrubbed — it is author-curated. You choose which fields it
  shows; the values you template land in the durable approval event (TEL
  retention). A field that must not persist simply isn't templated. (The scrubbed
  audit `payloadSummary` is a separate, untouched layer.)
- One string, one language (Turkish for a Turkish operator). Per-language maps
  aren't a thing yet — one operator, one card.

## Expression grammar

Precedence, low → high:

```
or          := and ( ("or" | "||") and )*
and         := not ( ("and" | "&&") not )*
not         := ("not" | "!") not | comparison
comparison  := additive ( CMP additive )?      # CMP: == != > >= < <= matches contains in
additive    := multiplicative ( ("+"|"-") multiplicative )*
multiplicative := unary ( ("*"|"/") unary )*
unary       := "-" unary | primary
primary     := number | string | true | false | null
             | path                            # ident ("." ident | "[" INT "]" | "[" "*" "]")*
             | helper "(" args ")"
             | "(" or ")"
```

### Paths: dots, `[N]`, `[*]`

- `payload.fields.DocCurrency` — dotted field access.
- `payload.fields.DocumentLines[0].BaseEntry` — array element by index.
- `payload.fields.DocumentLines[*].LineTotal` — **projection**: collects that
  field across every element into a flat array. Elements where the field is
  missing/null are **dropped** (so `count(lines[*].X)` means "how many lines
  carry X"), and nested arrays flatten one level.

### Totality — the rule never throws, and what that implies

A missing path, a bad index, a non-array under `[*]`, a non-numeric operand, or
a division by zero all evaluate to `undefined` — and **no comparison matches
`undefined`**, so the rule simply doesn't fire. This is deliberate (a malformed
rule must not take the guard down) but it has an authoring consequence: **a
check over a field the request may omit silently turns itself off.** If your
boundary depends on a field being present, write the presence check as its own
rule (`reject_if: "not payload.fields.Quantity"` style, or a projection-count
comparison) — don't rely on the value check alone.

Same trap with aggregates: `sum(...)` over a projection where no element carries
the field is `sum([])` = **0**, and 0 is under every threshold. Pair a threshold
rule with a presence floor when the job demands it.

### Helpers

| Helper | Semantics |
|---|---|
| `count(x)` / `length(x)` | array length or string length; **0 for missing** |
| `contains(x, v)` / `x contains v` | array membership (loose: `2 == "2"`) or substring |
| `v in x` | same as `contains(x, v)` with operands flipped |
| `startsWith(x, s)` / `endsWith(x, s)` | string prefix/suffix; existential over an array |
| `matches(x, pattern)` / `x matches pattern` | **GLOB, NOT REGEX** — `*` = any run, `?` = one char, case-insensitive, anchored both ends. `.*10$` does NOT mean "ends in 10" — it means "starts with a literal dot, ends with a literal dollar sign," which matches nothing real. For suffix checks use `endsWith`; for domain checks use `"*@example.com"`. Existential over an array (any recipient matches). |
| `sum(x)` | sum of a numeric array (a `[*]` projection); empty ⇒ 0; any non-numeric element ⇒ `undefined` |
| `abs(x)` | absolute value; non-numeric ⇒ `undefined` |
| `lower(x)` / `upper(x)` | case fold |

### Arithmetic + comparison semantics

- Arithmetic is **strictly numeric**: `null`/missing never coerce to 0, there is
  no string concatenation, `/0` is `undefined`.
- Ordering comparisons (`> >= < <=`) coerce both sides with `Number(...)` — so a
  numeric **string** compares fine (HANA returns DECIMALs as strings like
  `"28.000000"`; `payload.x > preflight.inspect` works).
- `==`/`!=` are strict — `2 == "2"` is **false**. Use them on same-typed values.
- Numbers tolerate decimals (`* 1.1` for a +10% threshold). Negative literals
  work via unary minus.

## `when:` — scope a rule to the requests it applies to

```yaml
when: "payload.fields.Lines[0].LinkedOrderId"     # only documents that reference an order
```

`when:` is evaluated **before** the preflight, on `payload`/`request`/`context`
only (`preflight.*` is rejected at load). False ⇒ the rule does not apply at
all: no preflight fetch, no verdict. Use it whenever a rule's preflight is only
*meaningful* for a subset of requests — e.g. a check that fetches a linked
document applies only when the link field is present; without `when:`, the fetch
would 404 on every other request and `on_preflight_error` would block traffic
the rule was never about.

## `preflight:` — judge the request against live upstream state

A rule can fetch state from the governed system mid-decision and compare the
request against it (does the referenced document exist? what is its total? is
this a duplicate?):

```yaml
preflight:
  method: GET
  path: "/b1s/v2/Things?$filter=Code eq '{payload.fields.Code}'&$select=DocEntry"
  inspect: response.value          # expression over { response, payload } → preflight.inspect
  timeout_ms: 5000                 # optional
```

- **`path` is the RAW upstream path** — include the system's API prefix (SAP:
  `/b1s/v2/...`), unlike `on.path` which is canonical. `path_from: request.path`
  replays the incoming path instead.
- **`{payload.x.y}` templates** substitute decoded-body values into the path.
  Substituted values are escaped for you (quote-doubling + percent-encoding) —
  write the template naturally inside an OData string literal
  (`eq '{payload.fields.Code}'`); never try to pre-escape.
- **Array fields template by dotted index** — `{payload.fields.DocumentLines.0.BaseEntry}`
  (dots, not brackets, inside `{…}` templates).
- `inspect` runs over `{ response, payload }`; its result becomes
  `preflight.inspect` for `reject_if`/`approve_if`. Index/projection work here
  too (`response.rows[0].TOTAL`, `response.value`).
- **A failed fetch never fails open.** Non-2xx → the rule's
  `on_preflight_error` verdict (`reject` default, or `needs_approval` to send it
  to a human instead).

## `examples:` stubs for preflight rules

(`guards.md` covers why examples exist; this is the stub mechanics.) A matched
preflight rule in an example needs its fetched state pinned:

```yaml
examples:
  - name: …
    request: { method: POST, path: /b1s/v2/Things, body: '{…}' }
    given:
      preflight: { value: [] }            # shared stub: any preflight rule reads this
      preflights:                          # per-rule stubs, keyed by RULE NAME —
        rule-a: { value: [{ DocEntry: 1 }] }   # required when several preflight
        rule-b: { rows: [{ TOTAL: "0.00" }] }  # rules match the same request
      # preflightError: "upstream 503"     # instead: force the fetch to FAIL
    expect: reject
```

Per-rule stubs override the shared one; a `preflights:` key naming no preflight
rule fails the example (typo guard); a matched preflight rule with no stub at
all fails loudly. A rule whose `when:` gate is false needs **no** stub — the
fetch never happens. Make stub values **realistic** (if the live system returns
DECIMAL strings, stub `"0.000000"`, not `0` — the example then proves the
coercion too).

## Authoring discipline — from job spec to rules

1. **One business rule = one named rule**, with the business language in
   `reason:` (it is what the employee and the operator read on a 403).
2. **Verify field semantics against live data before writing the check.**
   Systems use sentinel values, not absence: a field you expect missing may
   arrive as an explicit `-1`, `""`, or `"N"`. A presence check
   (`count(lines[*].Field)`) silently never fires against a sentinel — match the
   real values instead (`lines[*].Field contains 22`). Pull one real document
   through the employee's read path, or ask the operator for one, FIRST.
3. **Write the examples from the job spec, not from the rule.** State the
   intent as concrete request → verdict pairs ("this body must reject") before
   or while writing the expression — `ansur guards validate` then catches the
   rule disagreeing with the intent.
4. **Cover both directions**: the case that must block AND the nearest legal
   case that must pass (the over-blocking bug is as real as the under-blocking
   one).
5. **A rule that ignores its input is flagged.** `validate` probes each rule
   with generated inputs; one whose verdict never changes (the glob-footgun
   class) fails. A deliberately constant rule must be the literal
   `reject_if: "true"` / `approve_if: "true"` — that form is allowlisted.
6. **Thresholds are policy knobs.** Put the number in the expression
   (`> preflight.inspect * 1.1`) with a comment naming the business meaning —
   the customer tunes it by editing the repo, no platform change.
7. Loop: author → `ansur guards validate` → fix → push → watch
   `ansur trace` audits under `observe`/`gated` → tighten.

---

## System-specific notes — SAP only

Everything above applies to every guard-system. The two features below exist
only on the SAP write guard (`sap-service-layer/`).

### Cross-upstream preflight: `upstream: sap-read`

When a check needs data the write API can't answer in one call (joins,
aggregates across documents), the rule can run **one server-side SQL** against
the tenant's HANA read backend instead of multiple OData fetches:

```yaml
preflight:
  upstream: sap-read
  method: POST
  path: /query
  body: '{"sql":"SELECT SUM(T.\"Quantity\") AS \"TOTAL\" FROM SOME_TABLE T WHERE T.\"DocEntry\" = {payload.fields.DocumentLines.0.BaseEntry}"}'
  inspect: response.rows[0].TOTAL
```

- The response shape is `{ ok, columns, rows: [ {COL: value} ], rowCount }`;
  DECIMAL values arrive as strings — ordering comparisons coerce, `==` does not.
- `body` templates escape substituted values for both SQL and JSON — write
  templates naturally, never pre-escape.
- Requires the tenant's `sap-hana` read backend to be provisioned
  (`sap-hana.users` secret — see `sap.md`); the query runs as the connector
  config's `preflightReadRole`, and **that HANA user must have SELECT on the
  tables the SQL touches**, or the preflight fails closed. Validate the SQL
  through the employee's read path first (same `POST {sql}` shape).
- Only SELECT/WITH statements are accepted; quote identifiers (`T."Quantity"`).

### Batched writes (`$batch`)

OData `$batch` envelopes are unbundled automatically: every sub-request is
evaluated against the rules **as if sent alone**, and the worst verdict decides
the whole batch (any reject ⇒ 403; else any approval ⇒ hold; else forward). A
malformed envelope is rejected outright. You do not write batch-specific rules —
cover the entities, and batches are covered.
