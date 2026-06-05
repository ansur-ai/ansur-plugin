# Reading an agent's trace

`ansur trace <agent>` pulls a snapshot of one agent's recent activity as folded
NDJSON to `.ansur/traces/<agent>-<ISO>.ndjson`. Use it to see what the employee
actually did, then decide the next edit.

```bash
ansur trace bdr                       # recent activity → a file under .ansur/traces/
ansur trace bdr --since 2h            # last 2 hours
ansur trace bdr --until 2026-06-01T18:00:00Z --limit 50
```

stdout is **one summary line** (`wrote N records … M turns, K audit, from–to`).
The records go to the file.

> **Never `cat` the trace into context** — it can be large. It's a file you
> `jq`/`grep`. (`.ansur/` is auto-gitignored; pulling never dirties the repo.)

## Two record kinds

**`turn`** — one agent turn span. Key fields:
- `turnId`, `threadId`, `agentId`, `bundleVersion`, `model`, `provider`
- `startedAt`, `endedAt`, `outcome` (a turn-close reason, or `"open"`)
- `input` `{ type, text, eventSeq }` — what triggered the turn
- `reply` `[{ messageId, text }]` — what the employee said
- `reasoning?` — the reasoning trace, if present
- `toolCalls[]` — each `{ name, path, status, input, result?, error?, latencyMs, callSeq, category? }`; `status` ∈ `pending | completed | errored | blocked`
- `failure?` `{ category, phase, message, retryable, … }` — present iff the turn errored. **This is the turn-diagnosis surface.**
- `assetsCommitted?` `{ commitSha, pathsChanged[] }` — memory/skills the employee wrote this turn
- `perf?` — per-turn timing

**`audit`** — a guard verdict (pass-through, already scrubbed): `{ kind:"audit",
source, type, ts, agentId, threadId, policyVersion?, payload }`. This is what a
guard allowed or blocked at the wire. `policyVersion` is the policy commit
(`GUARD_POLICY_REF`) that produced the verdict — answers "which policy version
allowed this call" (guard-audit + approval records carry it; cron audit does not).

## Recipes (jq the file, don't read it)

```bash
F=.ansur/traces/bdr-<ISO>.ndjson

# What did the employee say, and was the turn OK?
jq 'select(.turnId) | {at:.startedAt, outcome, in:.input.text, said:[.reply[].text]}' "$F"

# Only failed turns + why (the diagnosis surface)
jq 'select(.failure) | {at:.startedAt, phase:.failure.phase, category:.failure.category, msg:.failure.message}' "$F"

# Every tool call + status + latency
jq 'select(.toolCalls) | .toolCalls[] | {name, status, latencyMs, error}' "$F"

# Blocked tool calls (a hook or guard stopped it)
jq 'select(.toolCalls) | .toolCalls[] | select(.status=="blocked")' "$F"

# Guard verdicts at the wire
jq 'select(.kind=="audit") | {ts, type, policyVersion, payload}' "$F"

# Guard approval queue (needs_approval / approve / deny)
jq 'select(.kind=="audit" and (.type | test("approval"))) | {ts, type, payload}' "$F"

# What the employee wrote to memory/skills
jq 'select(.assetsCommitted) | {at:.startedAt, .assetsCommitted}' "$F"
```

## Using it

- **Wrong answer?** Pull the turn, read `reasoning` + `toolCalls`. Missing a fact
  ⇒ seed `memory/`. Missing a step ⇒ tighten `prompt.md` or add a `skill`.
- **A call was `blocked`?** A hook or guard stopped it — check `references/hooks.md`
  / `references/guards.md` and the matching `audit` record.
- **Expected Approve/Deny buttons but got none?** Check bundle `approvals.notify`,
  `channel bind`, guards `mode: gated` (not `enforced`), and audit records for
  `approval-requested`. Operator can still type `approve` / `deny` in chat.
- **A turn `failure`?** Read `failure.category` + `phase` to see which boundary
  broke (model, tool, guard, …) before assuming the model is at fault.

Then loop: **author → ship → trace**.
