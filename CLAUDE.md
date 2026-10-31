# CLAUDE.md — pi-ansible

Ansible playbooks that rebuild Ethan's two Raspberry Pis from a fresh SD card.
It's a learning project: explain Ansible concepts in plain English as you go.
See README.md for layout, usage and the roadmap.

## Rules
- **Test on the Lima VM first** (`-i inventory/test.yml`), then `--check --diff`
  against a real Pi, and only then apply for real. The Pis run live services.
- **Never wipe the Pi 500's SD card** to test: the Minecraft worlds have no
  backups. Real-hardware tests use a spare card or USB stick.
- **Pi 400:** never run a bare `docker compose up -d`; it would start the
  sleeper bot and post to a real Discord. Target services by name.
- Every role must be idempotent: the second run on the VM must be `changed=0`.
- `ansible-lint` must pass (production profile).
- No secrets in the repo until they're in ansible-vault (Phase 2).
- job-agent's unit files stay in the job-agent repo; the role copies them from
  the checkout. Don't duplicate them here.
