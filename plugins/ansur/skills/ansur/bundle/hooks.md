# Hooks — gates on the employee's agent loop

A bundle hook is a **bash script** the bundle ships under `hooks/`. At each
agent-loop boundary the daemon runs the matching script, speaking the same wire
protocol as Claude Code hooks. Hooks shape **how the employee works** — they are
fast, deterministic, in-process gates on the employee's *intent*.

> **Gates run on the daemon; the `post-tool` observer runs in the sandbox.**
> `pre-tool` and `stop` deny / rewrite / block, so they always run **on the
> daemon** with daemon privileges — a gate the untrusted sandbox runs is not a
> gate. `post-tool` cannot deny, so in pooled mode it runs **inside the
> sandbox** with `cwd=/workspace` — where the agent's `edit`/`write` land — so a
> lint/typecheck/test hook sees the actual edited file. Its message is injected
> back as lowest-tier user text: it advises, never commands. None of this is a
> security boundary; the load-bearing boundary on a real external side effect is
> the **guard** (`guards/guards.md`), not a hook.

## Where scripts live

```
hooks/
├── pre-tool/    # before a tool runs   — allow, rewrite the input, or deny
├── post-tool/   # after a tool ran      — inject feedback (CANNOT deny)
└── stop/        # when the agent wants to end the turn — allow or deny
```

One script per file. The file **must have the executable bit** (`chmod +x`).
`.gitkeep`, non-executable files, subdirectories, and symlinks are ignored.
Scripts in a dir run in lexical filename order; if several match a phase, the
**first `deny` wins**.

## Wire protocol

```
stdin   ← one-line JSON context (per-phase shape below)
stdout  ← one-line JSON decision (empty stdout on exit 0 = allow / no effect)
stderr  ← human-readable; on a deny it becomes the message shown to the model
exit 0  → stdout is parsed as the decision
exit 2  → deny — stderr becomes the deny message
other / timeout / malformed JSON / over output cap → fail-closed (denied)
```

Limits: **5s** wall-time, **64 KB** output cap. Exceed either and the script is
killed and **fails closed** — don't call slow network APIs from a `pre-tool` gate.

Runtime: `bash`, `jq`, `git`, `ripgrep`, `python3`, and plain `node` are on
`PATH` (daemon and sandbox share one image). **`tsx` is not** — write hooks in
bash (+ `jq`/`python3`) or call plain `node` on a `.js` file.

### Per-phase context (stdin) → decision (stdout)

**pre-tool** — `{ "phase":"preToolUse", "toolName":"...", "toolInput":{...} }`
→ `{"kind":"allow"}` · `{"kind":"allow","input":{...}}` (rewrite the tool input) ·
`{"kind":"deny","message":"why"}`.

**post-tool** — `{ "phase":"postToolUse", "toolName":"...", "toolResult":{...}, "ok":true|false }`
→ a `PostToolEffect[]`: `[{"kind":"message","message":"feedback to the model"}]`
or `[]` / empty. **Cannot deny** — the tool already ran.

**stop** — `{ "phase":"stop" }`
→ `{"kind":"allow"}` (let the turn end) or
`{"kind":"deny","message":"keep going because…"}` (fed back; the loop re-enters).

## Examples

Gate a write tool until it's confirmed (pre-tool):
```bash
#!/usr/bin/env bash
set -euo pipefail
input=$(cat)
tool=$(printf '%s' "$input" | jq -r '.toolName')
op=$(printf '%s' "$input" | jq -r '.toolInput.operation // ""')
if [[ "$tool" == "sap.write" && "$op" != "confirmed" ]]; then
  echo "sap.write requires an explicit confirmation step" >&2
  exit 2
fi
exit 0
```

Nudge the model after a failed tool (post-tool):
```bash
#!/usr/bin/env bash
set -euo pipefail
ok=$(cat | jq -r '.ok')
if [[ "$ok" == "false" ]]; then
  echo '[{"kind":"message","message":"tool failed — check the broker logs"}]'
fi
exit 0
```

Don't let the agent finish while tests are red (stop):
```bash
#!/usr/bin/env bash
set -euo pipefail
if ! pnpm run test >/tmp/t.log 2>&1; then
  echo "tests are failing — fix them before finishing" >&2
  exit 2
fi
echo '{"kind":"allow"}'
exit 0
```

## Hook or guard?

- **Hook** — fast, deterministic, shapes *how the agent works* (require a step,
  reorder, retry, block an obviously-bad call early). In-process, bypassable by
  the daemon.
- **Guard** — the non-spoofable boundary on the *actual external side effect* (the
  real `sap.write`, the outbound email). Human-in-the-loop approval and load-
  bearing policy belong here.

If it must not be bypassable, it's a guard, not a hook.

## Gotchas

- **`post-tool` only fires after a real tool call returns** — a pure-text /
  prompt-knowledge turn has nothing to fire after.
- **A `post-tool` hook sees the workspace via `cwd=/workspace`** in pooled mode
  (it runs in the sandbox). Resolve edited files by relative path; don't hard-code
  a daemon-side absolute path.
- **`post-tool` effects aren't persisted to the trace** — they're injected into
  the live conversation. Verify a post-tool hook by a side effect or an echo
  token, **not** by grepping the trace.
- **Live at the next idle turn, not mid-turn** — hooks reload with the bundle: a
  push rebuilds the agent at the new commit on its next idle access, re-binding
  hooks (they're cached per bundle version). A hook added during a turn isn't
  active until that rebuild — but no restart is needed.
- **Test before shipping:**
  ```bash
  echo '{"phase":"preToolUse","toolName":"sap.write","toolInput":{"operation":"draft"}}' \
    | ./hooks/pre-tool/your-hook.sh; echo "exit=$?"
  ```
- **`exit 0` + garbage stdout = fail-closed deny**, not a lenient allow. Emit
  valid JSON or nothing.

> Wire-protocol summary: exit 0 with valid JSON (or empty stdout) is honored;
> a non-zero exit, unparseable stdout, or a timeout is a **fail-closed deny** on
> gate phases — never a lenient allow. When in doubt, emit nothing and exit 0.
