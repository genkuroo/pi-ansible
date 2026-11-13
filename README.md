# pi-ansible

Ansible playbooks that rebuild my Raspberry Pis from a freshly flashed SD card
with one command, instead of setting them up by hand over SSH.

- **Pi 500** (`pi500`, over Tailscale): job-agent, my self-hosted job-search agent
  (private repo; systemd user timers), and a modded Minecraft server
- **Pi 400** (`homelab`): the Docker Compose stack behind gen-kuro.com

## Why

Both Pis were configured by hand, so a dead SD card meant rebuilding from
memory. Now the setup lives in code: it's documented, reviewable, and
repeatable. Ansible describes the **desired state** ("chromium is installed",
"this file is mode 600") and only changes what doesn't match, so the playbook is
safe to run any number of times (**idempotent**).

## Layout

```
ansible.cfg
inventory/
  hosts.yml          # the real Pis, grouped by what they run
  test.yml           # a throwaway VM standing in for a fresh Pi
  group_vars/pis.yml
  host_vars/pi500/
    main.yml         # settings + the NAMES of every secret (plain text)
    vault.yml        # the secret VALUES (ansible-vault, AES-256)
    contact.yml      # phone/email for letters (vault-encrypted AND gitignored)
playbooks/
  site.yml           # the one command
  ping.yml           # connectivity smoke test
roles/
  base/              # packages + timezone, every Pi
  tailscale/         # install from Tailscale's apt repo, join with a one-time key
  jobagent/          # Ansible version of job-agent's deploy/install.sh,
                     # plus its secrets and the board's domain
```

## Usage

```sh
python3 -m venv .venv && .venv/bin/pip install ansible-core ansible-lint
source .venv/bin/activate
# The vault password lives outside the repo (ansible.cfg points here):
#   ~/.config/pi-ansible/vault-pass   (mode 600, keep a copy in a password manager)

ansible-playbook playbooks/ping.yml                 # can we reach everything?
ansible-playbook playbooks/site.yml --check --diff  # dry run: what WOULD change
ansible-playbook playbooks/site.yml                 # apply
```

## Rebuilding a Pi

1. Flash Raspberry Pi OS Lite 64-bit with **Raspberry Pi Imager**, and set the
   hostname, user, Wi-Fi and SSH public key in its customization dialog. (On
   Trixie, a hand-written `custom.toml` is silently ignored.)
2. Clear the old host key: `ssh-keygen -R pi500.local`
3. In the Tailscale admin console, delete the old `pi500` machine (so the new
   one gets the same name) and make a one-time auth key.
4. On the home network, the new card isn't on Tailscale yet, so reach it by
   its LAN name for this first run:
   ```sh
   ansible-playbook playbooks/site.yml --limit pi500 \
     -e ansible_host=pi500.local -e tailscale_authkey=tskey-auth-...
   ```
   That's it: no secrets to copy by hand. The playbook joins the tailnet, writes
   the deploy key, env file and Google key from the vault, starts the timers,
   and re-creates the board's DNS record, certificate and Caddy setup.

Ansible restores **configuration**, not **data**. Databases, letters and
Minecraft worlds need backups of their own.

## Testing on a throwaway VM

Raspberry Pi OS is Debian 13 plus Pi extras, and Apple Silicon is arm64 like
the Pi, so a Debian 13 VM under [Lima](https://lima-vm.io) is a close stand-in
for a fresh card:

```sh
limactl create --name=testpi --plain --cpus=2 --memory=2 --disk=12 -y template:debian-13
limactl start testpi
git -C ../job-agent bundle create "$PWD/.cache/job-agent.bundle" main
ansible-playbook -i inventory/test.yml playbooks/site.yml   # run twice: 2nd = changed=0
limactl delete -f testpi                                    # wipe, then repeat
```

`--plain` turns off Lima's home-folder mount, so the VM really is blank. The VM
has no GitHub deploy key, so the test inventory sets `jobagent_repo_bundle` and
the role clones from a git bundle copied over from the Mac instead.

**Result (2026-10-31):** a blank VM is fully set up in one run (~100s, 0
failures), and the next run reports `changed=0`.

### What the VM caught

Neither of these shows up on the existing Pi, because it was set up by hand
and already had what was missing. Both would have broken a real rebuild:

1. **The git module can't clone from a bundle.** It turns a path into a
   `file://` URL, and git can't read a bundle that way. Fix: unpack the bundle
   into a bare repo first, re-unpacking only when the bundle changes, so the
   second run stays `changed=0`.
2. **The pip module needs `python3-packaging` on the target.** That's not for
   job-agent; Ansible's own module needs it, and a fresh OS doesn't ship it.
   Fix: install it in the role's package list.
3. **The env template hardcodes `/home/ethan`.** Fine on the real Pi, but under
   any other username the board page crashed on startup trying to write there.
   Fix: right after creating the env file from the template, point its paths at
   the real home folder. A file that already exists (with secrets) is never touched.

## Running it on the real Pi 500

**Result (2026-11-10):** `--check --diff` first, then applied for real: 5 changes,
0 failures, and a second run reported `changed=0`. job-agent, the board page and
Minecraft kept running throughout.

What the real Pi caught:

1. **The role had drifted behind job-agent.** job-agent had since added a board
   page (`jobagent-web.service`) and two timers that need no secrets. A rebuild
   would have installed them but left them off. Fix: the role now always enables
   them, and restarts the board page whenever a pull brings new code.
2. **A dry run can't evaluate the pip task on a fresh OS.** The library it needs is
   only "would install" in check mode. Fix: that one task ignores errors in check
   mode only (`ignore_errors: "{{ ansible_check_mode }}"`); real runs fail normally.
3. **A hand-written SSH entry would have been duplicated.** Ansible only recognizes
   lines inside its own `BEGIN/END ANSIBLE` markers. Fix: backed up `~/.ssh/config`,
   removed the hand-written entry, and let Ansible own the only copy.

## Phase 2: secrets in ansible-vault

Every secret job-agent needs (Claude token, Discord webhooks, healthcheck URLs,
Google service-account key, GitHub deploy key, Cloudflare DNS token) is now in
`vault.yml`, encrypted with AES-256. The repo can be public because the
ciphertext is useless without the password, and the password file is outside
the repo (`ansible.cfg` points at `~/.config/pi-ansible/vault-pass`).

How it's put together:

- **Names in plain text, values in the vault.** `host_vars/pi500/main.yml` maps
  readable names (`jobagent_secrets.DISCORD_WEBHOOK_SRE`) to vault variables
  (`vault_jobagent_env_discord_webhook_sre`). You can grep for where a secret
  is used without decrypting anything, and the `vault_` prefix says where every
  value comes from. This is the layout Ansible's own docs recommend.
- **Contact details are encrypted and kept off GitHub.** Phone and email are in
  `contact.yml`, which is vault-encrypted *and* gitignored. A token can be
  rotated if the vault password ever leaks; a phone number can't.
- **Nothing secret is ever printed.** Every task that handles a secret has
  `no_log: true` (hides values in output and logs) and `diff: false` (otherwise
  `--diff` would show the old and new file side by side). Tested: a run at
  `-vvv --diff` that changed the env file printed none of its values.
- **The vault is the source of truth.** The env file is rebuilt from a template
  every run, so a hand edit on the Pi gets put back. To rotate a token:
  `ansible-vault edit inventory/host_vars/pi500/vault.yml`, then run the playbook.
- **Fail early, not halfway.** The role first checks every secret it needs is
  set, so a missing `contact.yml` or a wrong password stops the run before
  anything is touched, instead of writing an env file full of blanks.
- **The test VM gets fake secrets.** `inventory/test.yml` sets obviously fake
  values (`https://discord.invalid/...`), so the vault code path is tested without
  the VM holding real tokens or posting to the real Discord.
- **Tailscale auth keys are passed at run time, not stored.** They expire after
  90 days at most, so one saved in the vault would be dead when a card actually
  fails. `-e tailscale_authkey=...` on the rebuild run instead.
- **The board domain reuses job-agent's script.** `jobs.gen-kuro.com` (tailnet-only
  HTTPS via Caddy, Let's Encrypt DNS challenge through Cloudflare) is set up by
  job-agent's `deploy/setup-domain.sh`. The role writes its Cloudflare token
  from the vault and runs it only if the certificate or Caddy config is missing,
  so normal runs never touch DNS or request a new certificate. (The old roadmap
  said Tailscale Serve; the board had since moved to its own domain.)

**Result (2026-11-12):** the vault was checked against the live Pi first by
comparing SHA-256 hashes of each value, so nothing was displayed. Then:

- A blank VM was set up in one run (~86s, 0 failures), and its second run reported `changed=0`.
- A dry run on the real Pi 500 showed the env file, keys and domain setup all
  unchanged.
- The **first real apply was `changed=0`**. The env file rendered from the vault
  is byte-for-byte the hand-made one, so the vault provably holds what the Pi runs on.
- `https://jobs.gen-kuro.com` kept answering throughout.

Known dry-run noise: under `--check`, downloading Tailscale's signing key always
shows "changed" (Ansible can't compare without downloading), and so does pip on
a fresh OS. Real runs report `changed=0`.

## Roadmap

- [x] **Phase 1:** scaffold, inventory, `base` + `jobagent` roles, VM test loop
- [x] Dry run, then a real apply, against the Pi 500 over Tailscale (2026-11-10)
- [x] **Phase 2:** secrets in **ansible-vault** (env file, deploy key, Google key,
      Cloudflare token), a `tailscale` role (install + join with a one-time key),
      and the board's domain via job-agent's `setup-domain.sh` (2026-11-12)
- [ ] **Phase 3:** Minecraft role (Temurin 8, `minecraft` user, `radlite.service`,
      playit) plus world backups
- [ ] **Phase 4:** Pi 400 Docker stack (start services by name, never a bare
      `docker compose up -d`). Note: the `base` role has never run on the Pi 400;
      a dry run shows it would add vim + unattended-upgrades. Decide whether
      automatic upgrades belong on the Docker host first.
- [ ] **Phase 5:** real-hardware test: boot the Pi 500 from a spare SD card or USB
      stick and rebuild it
