# claude-auth

`cc-auth` — save, switch, and inspect multiple Claude Code authentication slots.

If you use more than one Claude account (personal/work, multiple Max plans, a shared org login), Claude Code only ever sees one set of credentials at a time. `cc-auth` snapshots those credentials into named slots and lets you swap between them, while caching each account's profile and rate-limit usage so you can see — at a glance — which account has headroom left.

## How it works

Claude Code stores its OAuth credentials in:

- **macOS**: the login Keychain, under service `Claude Code-credentials`
- **Linux**: `~/.claude/.credentials.json`

`cc-auth` reads/writes that one active location and keeps per-account copies under `~/.cc-auth/slots/<name>/`:

- `profile.json` — last response from `/api/oauth/profile` (email, tier, plan)
- `usage.json` — last response from `/api/oauth/usage` (5h / 7d utilization, reset times)

The actual OAuth credentials are stored separately — in the macOS login Keychain (service `cc-auth`, account = slot name), or in `credentials.json` alongside the metadata on Linux. See [Security](#security) for details.

Switching a slot just writes its `credentials.json` back into the active location. Restart any running Claude Code sessions to pick up the change.

## Install

Drop `cc-auth` somewhere on your `PATH` and make it executable:

```sh
install -m 0755 cc-auth ~/.local/bin/cc-auth
```

Requires Python 3.10+. `fzf` is optional — if present, `cc-auth switch` uses it for an interactive picker; otherwise it falls back to a numbered prompt.

## Commands

| Command | Description |
| --- | --- |
| `cc-auth save [<name>]` | Snapshot the active credentials into a slot. Defaults the name to the logged-in account email. |
| `cc-auth switch [<name>]` | Activate a slot. With no name, opens an `fzf` picker (or numbered prompt). |
| `cc-auth list` | List slots with cached email, tier, and 5h/7d utilization sparkbars. |
| `cc-auth current` | Print the slot name matching the currently active credentials. |
| `cc-auth delete <name>` | Remove a slot. |
| `cc-auth refresh [<name>]` | Re-fetch profile + usage for one slot, or all slots. |
| `cc-auth stats [<name>]` | Detailed cached usage table for one or all slots. |
| `cc-auth whoami` | Live profile + usage for the active account. |

## Typical workflow

```sh
# log in to account A via `claude`, then snapshot it
cc-auth save                    # auto-names from account email

# log in to account B, snapshot that too
cc-auth save work

# see who's got headroom
cc-auth list

# swap to whichever account has the most quota left
cc-auth switch
```

## Security

`cc-auth` never sends your credentials anywhere. Tokens only move between two stores on your own machine:

- **macOS**: every credential — both the *active* one Claude Code reads (service `Claude Code-credentials`) and every saved *slot* (service `cc-auth`, account = slot name) — lives in the login Keychain. Reads and writes go through the system `security` binary, so the OS prompts for Keychain access the first time and secrets are encrypted at rest under your login password. No OAuth tokens are ever written to disk by `cc-auth` on macOS.
- **Linux**: the active credential is `~/.claude/.credentials.json` and slot credentials are `~/.cc-auth/slots/<name>/credentials.json`, both written `chmod 600`. There's no system-wide secret store assumed; if you need stronger isolation, run on an encrypted volume (LUKS) or wrap with `pass`/`secret-tool`.

The only files `cc-auth` writes under `~/.cc-auth/slots/<name>/` are `profile.json` and `usage.json` — non-sensitive metadata (email, tier, rate-limit counters) used for the `list` / `stats` views.

The only network calls are authenticated `GET`s to `api.anthropic.com/api/oauth/profile` and `/api/oauth/usage`. No third-party services, no telemetry.

### Migrating from older `cc-auth`

Earlier versions stored slot credentials as `~/.cc-auth/slots/<name>/credentials.json` on macOS too. On first read, `cc-auth` transparently moves each legacy file into the Keychain and deletes the on-disk copy — no action required.

## Environment

- `NO_COLOR` — disables ANSI color output
- `CC_AUTH_FORCE_COLOR` — forces color even when stdout is not a TTY

## License

MIT — see [LICENSE](LICENSE).
