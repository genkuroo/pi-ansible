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
  host_vars/pi500.yml
playbooks/
  site.yml           # the one command
  ping.yml           # connectivity smoke test
roles/
  base/              # packages + timezone, every Pi
  jobagent/          # Ansible version of job-agent's deploy/install.sh
```

## Usage

```sh
python3 -m venv .venv && .venv/bin/pip install ansible-core ansible-lint
source .venv/bin/activate

ansible-playbook playbooks/ping.yml                 # can we reach everything?
ansible-playbook playbooks/site.yml --check --diff  # dry run: what WOULD change
ansible-playbook playbooks/site.yml                 # apply
```

## Rebuilding a Pi

1. Flash Raspberry Pi OS Lite 64-bit with **Raspberry Pi Imager**, and set the
   hostname, user, Wi-Fi and SSH public key in its customization dialog. (On
   Trixie, a hand-written `custom.toml` is silently ignored.)
2. Clear the old host key: `ssh-keygen -R pi500.local`
3. Put the secrets back (until Phase 2 below): the job-agent deploy key at
   `~/.ssh/jobagent_deploy` and `~/.config/jobagent/env`.
4. `ansible-playbook playbooks/site.yml`

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

## Roadmap

- [x] **Phase 1:** scaffold, inventory, `base` + `jobagent` roles, VM test loop
- [x] Dry run, then a real apply, against the Pi 500 over Tailscale (2026-11-10)
- [ ] **Phase 2:** secrets in **ansible-vault** (deploy key + env file), plus
      Tailscale itself (install, auth key, and Serve for the board page)
- [ ] **Phase 3:** Minecraft role (Temurin 8, `minecraft` user, `radlite.service`,
      playit) plus world backups
- [ ] **Phase 4:** Pi 400 Docker stack (start services by name, never a bare
      `docker compose up -d`)
- [ ] **Phase 5:** real-hardware test: boot the Pi 500 from a spare SD card or USB
      stick and rebuild it
