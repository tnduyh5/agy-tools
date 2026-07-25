# agy-tools

![platform](https://img.shields.io/badge/platform-macOS-lightgrey)
![license](https://img.shields.io/badge/license-MIT-blue)
![homebrew](https://img.shields.io/badge/homebrew-tnduyh5%2Ftap-F9A03C)

Two tiny command-line helpers for the **Antigravity CLI** (`agy`, Google's
Gemini coding agent) on **macOS**:

| Command | What it does |
| --- | --- |
| `agy-account` | Log into several accounts once, then switch between them with a single command — no browser re-login. |
| `agy-usage` | Show the remaining model quota for **all** your saved accounts at once, without switching the live login. |

![demo](assets/demo.svg)

Both are read-only against your macOS keychain except for the explicit
`agy-account save` / `use` / `logout` actions. No secrets are stored in this
repo (see [How it works](#how-it-works)).

---

## Why

`agy` keeps exactly one login at a time, stored as an OAuth token in the macOS
login keychain. If you have more than one account (e.g. two Google AI Ultra
subscriptions) the only built-in way to switch is to log out and re-authenticate
in the browser every time — and there is no command to see how much quota each
account has left.

`agy-account` snapshots each account's token under its own name and swaps them in
place, so switching is instant. `agy-usage` refreshes each saved token *offline*
and calls the same quota endpoint the CLI uses, so you can compare accounts at a
glance.

---

## Requirements

- **macOS** (uses the `security` keychain CLI — not portable to Linux/Windows)
- The **Antigravity CLI** installed and logged in at least once (`agy`)
- **Python 3** (standard library only — no `pip install`)
- Optional: the `agy` binary on your `PATH`, or at `~/.local/bin/agy`

---

## Install

### Homebrew (recommended)

```bash
brew install tnduyh5/tap/agy-tools
```

### From source

```bash
git clone https://github.com/tnduyh5/agy-tools.git
cd agy-tools
# put the two commands on your PATH (adjust the target dir to taste)
ln -sf "$PWD"/bin/agy-account "$PWD"/bin/agy-usage ~/.local/bin/
```

Make sure `~/.local/bin` is on your `PATH` (it already is if `agy` lives there).
Installing from source with symlinks means edits to the repo take effect
immediately, which is handy if you want to tweak the scripts.

---

## `agy-account` — switch accounts

```
agy-account save <name>   snapshot the CURRENT login as profile <name>
agy-account use <name>    switch the live login to profile <name>
agy-account list          list saved profiles
agy-account whoami        show the email agy last authenticated as
agy-account logout        delete the live token (next `agy` run opens browser login)
```

### First-time setup (two accounts)

```bash
# 1. You are already logged into account A. Save it:
agy-account save work

# 2. Drop the live token and log into account B in the browser:
agy-account logout
agy            # complete the browser login for account B
agy-account save personal
```

### Everyday use

```bash
agy-account use work       # instant switch, no browser
agy-account use personal
agy-account whoami         # who am I right now?
agy-account list           # what have I saved?
```

Notes:

- Every `use` first snapshots the currently-live token into a profile named
  `_last`, so a login you forgot to `save` is never lost.
- `agy` reads its token **at startup**. If an `agy` session is already running,
  restart it after switching. Avoid switching while a background `agy` job is
  running — the running session keeps the old token and you can mix up quota
  between accounts.

---

## `agy-usage` — see remaining quota for all accounts

```bash
agy-usage        # one line per account: lowest remaining bucket + reset time
agy-usage -f     # full per-model breakdown
```

Example:

```
work      you@gmail.com          Ultra  lowest gemini-2.5-flash 100%   reset 2026-07-26 09:21 UTC
personal  you.two@gmail.com      Ultra  lowest gemini-3-pro-preview 42% reset 2026-07-26 09:21 UTC
```

The "reset" time is a **rolling window** (now + the quota window), not a fixed
daily clock, so it drifts by a few seconds between calls — that's expected.

---

## How it works

- **Token storage.** `agy` stores its OAuth token in the macOS login keychain at
  `service=gemini`, `account=antigravity`. `agy-account` copies that blob to/from
  per-name entries under `service=gemini-profile`. Switching accounts is just
  overwriting the live entry — nothing is sent anywhere.

- **Reading quota without switching.** `agy-usage` reads each saved token's
  `refresh_token`, exchanges it for a short-lived access token against Google's
  OAuth endpoint, then calls `v1internal:loadCodeAssist` (to find the account's
  project) and `v1internal:retrieveUserQuota`. This never modifies the live login
  and never writes a token back to the keychain.

- **No secrets in this repo.** Refreshing a token needs the OAuth
  *client_id/secret*. Rather than committing Antigravity's credentials, `agy-usage`
  extracts them from your **local `agy` binary** at runtime (they are public
  "installed-app" credentials embedded in it), verifies which pair works, and
  caches the winner in `~/.cache/agy-tools/oauth.json`. If Antigravity ever
  rotates them, delete that cache and the tool re-discovers them.

---

## Caveats

- **macOS only.** Everything hangs off the `security` keychain CLI.
- Reading a keychain secret may prompt for keychain access the first time,
  depending on your setup. Allow the `security` tool and it won't ask again.
- These tools are unofficial and depend on `agy`'s current keychain layout and
  internal API paths. A future `agy` release could change either and break them.

---

## Disclaimer

Unofficial, not affiliated with Google or the Antigravity team. Use it only with
your **own** accounts and within the applicable Terms of Service. It moves tokens
around your own machine and reads your own quota — nothing more.

## License

MIT — see [LICENSE](LICENSE).
