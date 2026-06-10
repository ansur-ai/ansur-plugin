# Initial setup (one-time)

Run this once per machine + per org. After it's done, return to the build loop in
`SKILL.md` — you won't need this file again. Re-running `login` is only needed
when the token expires.

## 0. Install the CLI and the plugin

- **CLI** — `@ansur-ai/cli` is on the public npm registry. Install the `ansur`
  binary globally:
  ```bash
  npm install -g @ansur-ai/cli@latest   # @latest avoids a stale-cache no-op on an old global
  ansur --version
  ```
  (Single bin `ansur`, no runtime deps.)
- **Endpoint — set this first.** The CLI defaults to `http://127.0.0.1:8080`
  (local dev). For a **hosted** platform you MUST point it at the real URL, or
  every command (`login` included) hits localhost and fails with a connection
  error. Set it once for the session:
  ```bash
  export ANSUR_ENDPOINT=https://<your-platform-host>   # e.g. https://demo.useansur.com
  ```
  (or pass `--endpoint <url>` per command).
- **Plugin** — this skill ships in the `ansur` Claude Code plugin. If you're
  reading this, it's already loaded.
- **`gh` (GitHub CLI)** — needed only for a **personal-account** GitHub link, to
  create the bundle + `<login>/guards` repos (org accounts auto-create — skip
  this). Ensure it's installed and authed to the customer's account before
  `bundle create` / `guards init`; if it's missing, install it, and if it's not
  logged in, run `gh auth login` and let the customer complete it:
  ```bash
  gh --version || brew install gh      # or the platform's installer
  gh auth status || gh auth login      # customer logs into their own account
  ```

## 1. Authenticate — `ansur login`

Device-code flow (RFC 8628): the CLI prints a verification URL + code, opens the
browser, and polls until you approve. The bearer token is cached in the config
dir (`~/.config/ansur/` by default), so later commands are already authed.

```bash
ANSUR_ENDPOINT=https://<your-platform-host> ansur login   # or export it (step 0)
```

If `login` errors with a connection refused at `127.0.0.1:8080`, the endpoint
isn't set — see step 0. Your email must also be on the platform's beta allowlist.

## 2. Create the tenant — `ansur init "<Org>"`

The first authenticated user with no tenant becomes its owner. This clears the
`no_tenant` 403 that every `/platform/*` call returns until a tenant exists.

```bash
ansur init "Acme Packaging"
```

Strictly **once per org**. After this, `ansur whoami` resolves to your tenant.

## 3. Link GitHub — `ansur github connect`

The platform needs write access to **host and clone the bundle repos**. That's
infrastructure the *platform* uses, not runtime access the *employee* uses — so
it has its own verb and is **not** a connector.

```bash
ansur github connect          # prints the Ansur GitHub App install URL, opens it, polls
ansur github status           # confirm the link (app | pat | none)
```

- **Tell the user to choose "Only select repositories" — NOT "All repositories."**
  GitHub auto-adds any repo the App *creates* to the installation, so each
  `bundle create` repo is covered (push + webhook) without granting access to
  every repo. If the install screen forces selecting ≥1 repo, any placeholder
  works — the created repo auto-joins regardless.
- One install covers **all** the customer's bundle repos and all future agents.
- **Fallback:** `ansur github connect --pat` accepts a fine-grained PAT (for orgs
  that forbid GitHub Apps). The PAT must allow repo Contents R/W.
- This must complete **before** the first `ansur bundle create`.

## 4. Sanity check

```bash
ansur whoami          # resolves token → customerId (no error = tenant OK)
ansur github status   # app/pat with an installation id = GitHub OK
```

If both pass, setup is done. Go build (`SKILL.md` step 1).

## External prerequisites (operator-side — may block the *live* path)

These are platform-deployment steps, not things you run as the meta-harness. If a
step in the build loop fails at the live boundary, it's likely one of these isn't
stood up yet on the target platform:

- **Gmail / OAuth connectors** need a platform-owned Google OAuth project +
  redirect URI on the live `/oauth/callback`. Until then `connector add gmail`
  can't complete real consent.
- **Telegram** needs `PLATFORM_BASE_URL` + `TELEGRAM_WEBHOOK_SECRET` set on the
  deployed daemon (and a real @BotFather token) before a bound bot receives.
- **GitHub App** must be registered with `GITHUB_APP_ID` / `GITHUB_APP_PRIVATE_KEY`
  / `GITHUB_APP_SLUG` / `GITHUB_WEBHOOK_SECRET` on the daemon. Without it, only the
  `--pat` fallback works.

If you hit one of these, tell the user it's an operator/deploy step — not
something the bundle author can fix.
