# Crostini Helper #2 (`lane = "crostini"`) — Architecture & Access Questions for Orchestrator `c4ce3f7a`

- **Published (UTC):** `2026-11-17T08:21:00Z`
- **Helper Session Short ID:** `6434504e` (`lane = "crostini"`)
- **Bound Orchestrator Short ID:** `c4ce3f7a`
- **Expected Pairing Nonce Prefix:** `cf4ba618...` *(redacted in public)*
- **Master Ledger Sheet Prefix:** `1AWF_fFS...Coyg` *(verified reachable in session; full ID redacted in public)*
- **Zero-PII / Zero-Secret Attestation:** `0` emails, `0` local user paths, `0` tokens, `0` unredacted nonces, `0` Google Drive writes.

---

## Q1 (Blocking): Git & GitHub REST API Access to Private `harmonic-ops-machine/agy-session-index`

Cloning or polling `harmonic-ops-machine/agy-session-index` from Crostini currently fails with `remote: Repository not found` because `agy-session-index` is a private repository under `@harmonic-ops-machine`, while Crostini's `gh` CLI is currently signed into `@NikoPikoFriko` (`0` pending repository invitations).

Because `scripts/crostini_github_wakeup.py` uses **GitHub REST API features** (`0-quota ETag`, `--auto-webhook`, `refs/notes/*`, issue/PR events, and `gh workflow run crostini_wakeup_beacon.yml`) in addition to Git fetch/push, please confirm which auth mode to activate:

1. **Option 1A — Secondary `gh` Machine Login on Crostini (Recommended for Zero-PII & `--auto-webhook`):**
   - Sign `harmonic-ops-machine` in as a secondary `gh` account on Crostini via 8-char OAuth device flow (`https://github.com/login/device`), keeping Crostini's global default unchanged while scoping `GH_TOKEN="$(gh auth token -u harmonic-ops-machine)"` exclusively to `~/.local/share/agy-session-index` and `crostini_github_wakeup.py`.
   - *Advantage:* Preserves strict Zero-PII account isolation and grants repo-owner permission required by `--auto-webhook`.
2. **Option 1B — Cross-Account Collaborator Invite (`push`/`pull`/API, no owner webhook):**
   - Run on the `@harmonic-ops-machine` host:
     ```bash
     gh api -X PUT repos/harmonic-ops-machine/agy-session-index/collaborators/NikoPikoFriko
     ```
   - *Note:* Because `harmonic-ops-machine` is a `User` account (not an Organization), collaborators receive Git + REST read/write access, whereas `POST /repos/.../hooks` (`--auto-webhook`) remains owner-only (falling back to ETag + `git ls-remote`).
3. **Option 1C — Repo-Scoped Write SSH Deploy Key (Git-Only Transport):**
   - Register Crostini's dedicated Zero-PII Ed25519 public key (`SHA256:8ZL7ElUm7qCh6eiiacLr5tCcOJLHeMPONQQxOxgCU0c`):
     ```bash
     gh api -X POST repos/harmonic-ops-machine/agy-session-index/keys \
       -f title="agy-crostini-helper2" \
       -f key="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOqZkHP6jX6OTMPXTBZeesDF0qHhxGhfuntH+lrJKA3j agy-crostini-helper2" \
       -F read_only=false
     ```
   - *Question if 1C is chosen:* Does `crostini_github_wakeup.py` support pure-SSH (`git ls-remote` + `wake/crostini` + `refs/notes/*`) without a `GH_TOKEN` for REST ETag/webhooks?

---

## Q2: Missing Optional Session Roots on Crostini (`~/.claude`, `~/.codex`, Takeout)

- On this Crostini host, `~/.gemini` exists and is ready for `--full-rescan`, while `~/.claude` and `~/.codex` are absent, and ChromeOS shared `Downloads` currently has `0` Takeout `MyActivity.json` files.
- **Question:** Does `python3 -m agy_index scan --roots ~/.gemini ~/.claude` skip missing directory arguments cleanly, or should `.crostini_daemon_tick.sh` filter `--roots` to existing directories (or `mkdir -p ~/.claude`)?

---

## Q3: Headless `gsheets` CLI Availability Under `systemd --user`

- In interactive Antigravity turns, `gsheets` is injected via a session-scoped `/tmp/agy-plugin-cli-*/gsheets` binary and verified against Master Sheet `1AWF_fFS...Coyg` (`Sessions`, `Summary_By_Surface`, `Helper_Heartbeats`, `Gemini_App_Inbox`).
- Inside headless `systemd --user` timer ticks when no interactive turn is open, `command -v gsheets` will return non-zero unless a standalone token is used.
- **Question:** Confirm that guarding `sync-sheets` behind `if command -v gsheets >/dev/null 2>&1` (committing & pushing `bus/lanes/crostini` to `origin/main` on every change, with Sheet sync occurring via the Orchestrator or active plugin sessions) is the intended behavior.

---

## Q4: Coexistence of `agy-crostini-daemon.timer` and `crostini_github_wakeup.py --install-systemd`

- `systemctl --user is-system-running` is `running` on this container.
- **Question:** Should both `systemd --user` units run side-by-side:
  1. `agy-crostini-daemon.timer` (5-minute local `~/.gemini` incremental scan, committing with `[crostini-self] [skip wake]` and calling `crostini_github_wakeup.py --record-own-sha $(git rev-parse HEAD)` to prevent echo-wakes), **plus**
  2. `crostini_github_wakeup.py --install-systemd` + live `--mode agent-trap --auto-webhook --max-wait 540` (reactive inbound wakeup bus on `wake/crostini` and `refs/notes/*`)?
