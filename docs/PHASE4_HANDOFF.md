# Phase 4 handoff (Pi 400 / homelab stack)

Paused 2026-11-16 mid-Phase 4 to move to another computer. Read this first,
then `README.md` and `CLAUDE.md`. Delete this file when Phase 4 is done.

## Before you start on the new computer

Two files never go to GitHub, and the vault can't be opened without them:

1. **The vault password:** `~/.config/pi-ansible/vault-pass` (mode 600). Copy
   it from your password manager, or over Tailscale with `scp`. Never paste it
   into a chat or a commit.
2. **`inventory/host_vars/pi500/contact.yml`:** encrypted but gitignored. It's
   already encrypted, so any copy method is fine. Without it, Pi 500 runs stop
   at the "Check every secret" task by design.

Then:
- `python3 -m venv .venv && .venv/bin/pip install ansible-core ansible-lint`
- SSH must reach both Pis: `pi500` as `ethan`, and the `homelab` alias
  (in `~/.ssh/config`, pointing at the Pi 400's tailnet name) as `genkuro`.
  Check with `ansible-playbook playbooks/ping.yml`.
- The VM test loop uses Lima, which runs on macOS and Linux. On Windows it
  needs WSL2 (or a different throwaway VM).

## Decisions already made

- **Automatic security updates on the Pi 400: yes.** Running the `base` role
  there installs `unattended-upgrades` (Debian security patches only; it
  doesn't touch Docker's repo and doesn't reboot) plus `vim`.
- **D&D backups go off-site to R2: yes** (step 1 below). Stock and fitness
  data still never leave the house.

## The plan and where it stands

### Step 1: D&D off-site backup (DONE 2026-11-17)

Branch `dnd-offsite-backup` in **homelab-pi** (commit `8818e9a`, pushed, not
merged, not deployed):
- `backup.sh`: each D&D database goes through sqlite3's backup API before
  archiving (it used to be a plain `tar` of the live files, which the script's
  own comments warn against). Archive name and layout are unchanged.
- `r2-sync.sh`: also uploads `dnd-*.tar.gz` to `dnd/` in the bucket.
- Tested on the Pi into a temp folder: same 4 files as the old archive, and
  both databases (`registry.db`, `campaigns/campaign.db`) pass
  `PRAGMA integrity_check`. Temp files removed.

**Done:** merged to `main` (`8818e9a`) and deployed to the Pi. The nightly run
on 2026-11-17 used the new code, and R2 holds 17 D&D snapshots under `dnd/`.
Restore drill passed 2026-11-17: the newest R2 copy was byte-identical to the
local one (same SHA-256), both databases passed `PRAGMA integrity_check`, and
table/row counts matched the live app exactly.

The original checklist, for reference:
1. Merge the branch to `main` and push.
2. On the Pi: `cd /opt/homelab && git pull`. The two unit files only changed
   their `Description=`, so copy them over and reload:
   `sudo cp systemd/ffta-r2-sync.* /etc/systemd/system/ && sudo systemctl daemon-reload`
3. Run it once now instead of waiting for the night:
   `sudo systemctl start homelab-backup.service`, then
   `./scripts/r2-sync.sh`, then `./scripts/r2-sync.sh verify`. Expect a `dnd/`
   listing next to `ffta/`.
4. Restore drill: download one `dnd/` archive back from R2, untar it, and run
   `PRAGMA integrity_check` on both databases (the same drill `ffta` passed
   on 2026-09-02).
5. Note in homelab-pi's vault note + the workspace `CLAUDE.md` row.

### Step 2: the Pi 400's secrets into the vault

- `.env` on the Pi has 21 keys (16 filled). Import them into
  `inventory/host_vars/homelab/vault.yml` **through a pipe, never printed**
  (the Phase 2 approach: `ssh ... | python | ansible-vault encrypt --output`).
- Check the vault against the Pi by comparing hashes, never by displaying values.
- **Trap:** `FFTA_AUTH_HASH` is a bcrypt hash with every `$` doubled for
  Compose. Store the raw line value exactly as it is in the file, or Caddy
  gets a truncated hash and returns a silent 401.
- **`DND_SECRET_KEY` is stable state.** If it changes, every player gets logged out.

### Step 3: a `homelab` role (replaces homelab-pi's `scripts/bootstrap.sh`)

Mirror what `bootstrap.sh` does, as Ansible:
- Docker from Docker's apt repo, matching the Pi exactly:
  `deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable`;
  user in the `docker` group; `docker/daemon.json` (pinned DNS) installed from
  the repo, restarting Docker only when it changes.
- Clone the 6 repos into `/opt` (all public, over https, owned by `genkuro`):
  homelab-pi → `/opt/homelab`, plus `stock-tracker`, `fitness-dashboard`,
  `dnd-campaign-tracker`, `sleeper-discord-bot`, `ff-trade-analyzer`.
- `.env` from a template, byte-identical to the Pi's (same trick as the Pi 500
  env file, so the first real apply proves the vault is right with `changed=0`).
- systemd units copied from the homelab-pi checkout (don't duplicate them in
  this repo); enable the 7 timers.
- `touch .healthcheck-report.json` **before** any container is created
  (otherwise Docker makes a directory there and `/health` breaks silently).
- Start services **by name, never a bare `docker compose up -d`**. Caddy is a
  local build (`docker compose build caddy`).
- Tailscale Serve: 443 → 127.0.0.1:5001 (stock), 8443 → :5002 (fitness),
  8444 → :5003 (fantasy).
- The 2 GitHub Actions deploy keys in `authorized_keys` with their forced
  commands (`scripts/deploy-staging.sh` / `deploy-prod.sh`). Public keys only,
  not secret.
- `noatime` is already set in `/etc/fstab`; at most, assert it.

### Step 4: fix the tailscale role for the Pi 400

The Pi 400 must keep `--accept-dns=false` (the fix for the 2026-09-10 and
2026-10-20 DNS outages). The role's `tailscale up` doesn't pass it, so add a
per-host variable (`tailscale_accept_dns: false` for homelab) used on join,
and ideally enforce it with `tailscale set --accept-dns=false` on every run.

### Step 5: test, then apply

Same rules as always: Lima VM with fake secrets (Docker in the VM; with fake
tokens, cloudflared and the sleeper bot fail harmlessly), then `--check --diff
--limit homelab`, then a real apply, then a second run with `changed=0`.
Watch that gen-kuro.com, dnd, fantasy and the tailnet dashboards stay up.

### Step 6: clean up leftovers on the Pi 400 (needs Ethan's OK at the time)

- `/opt/homelab/.env.bak.20260904220328` holds real secrets and is **not
  gitignored**, so `git add -A` there would publish them. Delete it once the
  vault holds the secrets.
- `~/.ssh/github-deploy-staging` (private key) on the Pi itself: looks like a
  leftover from creating the CI key. Confirm it isn't used, then remove it.
- `~/.ssh/authorizzed_keys` (typo file from June): delete.

## Working style

Ethan is learning Ansible. Go one step at a time: explain the step in plain
English, do it, show the result, and wait for an OK. Explain things the way
he'd present them to a recruiter (what it does, how it works, why).
