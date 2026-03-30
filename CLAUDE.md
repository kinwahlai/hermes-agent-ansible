# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo does

Ansible playbook to install [hermes-agent](https://github.com/NousResearch/hermes-agent) on Debian/Ubuntu (Oracle Cloud ARM / aarch64). Runs as the `hermesagent` system user. Systemd-managed. Tailscale serve for remote HTTPS access at port **8444**.

See `HERMES_AGENT_PLAN.md` for the full build plan — it contains the complete repo structure, task order, variable definitions, and patterns to copy from reference repos.

## Reference repos on this machine

- `~/dev_repo/AI-projects/paperclip-ansible` — structural template; copy most files from here
- `~/dev_repo/AI-projects/openclaw-ansible` — defines what is already installed on the server

## Common commands

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# Lint
ansible-lint playbook.yml

# Dry run
ansible-playbook -i inventory playbook.yml --check --ask-become-pass

# Full run
ansible-playbook -i inventory playbook.yml --ask-become-pass

# Verify after deploy
systemctl status hermesagent
curl http://127.0.0.1:<hermes_port>/health
journalctl -u hermesagent -n 50

# CI test (Docker)
bash tests/run-tests.sh
```

## Critical constraints

**Do NOT reinstall or conflict with anything already on the server:**
- Node.js 22.x, pnpm, npm, Docker CE, Tailscale, Ollama, git, curl, build-essential, fail2ban, UFW

**Ports already in use — hermesagent must not use these:**
- 3000 (OpenClaw), 3100 (Paperclip), 8443 (Paperclip Tailscale), 11434 (Ollama)
- Use **8444** for Tailscale serve

**Existing system users:** `openclaw`, `ollama`, `paperclip`

**UFW:** Active with default-deny inbound. Do NOT add UFW rules. Keep hermesagent port localhost-only.

## Before writing install.yml

Inspect the install script first — do not pipe to bash:

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | cat
```

Confirm: binary path, aarch64 support, run command, default port, whether it tries to install Node.js.

## Architecture

Task order (must not change): `prereqs.yml → user.yml → install.yml → service.yml → tailscale.yml`

- `service.yml` and `tailscale.yml` are skipped when `ci_test: true`
- `prereqs.yml` must stay minimal — most deps are already present from openclaw-ansible
- `install.yml` must be idempotent: use `stat` check on binary path as guard before running installer
- The installer runs as `hermesagent` user via `become_user`, with `HOME` set to `/home/hermesagent`
- Systemd unit uses security hardening matching openclaw/paperclip pattern (`NoNewPrivileges`, `PrivateTmp`, `ProtectSystem=strict`)
