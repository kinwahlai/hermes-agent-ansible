# Agent Guidelines

## Project Overview

Ansible playbook for automated installation of [hermes-agent](https://github.com/NousResearch/hermes-agent)
on Debian/Ubuntu Linux (aarch64/x86_64). hermes-agent is a Python-based AI assistant from NousResearch
managed by uv, running as a systemd service, exposed via Tailscale serve.

## Key Principles

1. **Localhost only**: Port binds to 127.0.0.1. Access via Tailscale serve (port 8444) only.
2. **Python + uv**: hermes-agent is installed via its official `install.sh` (uses uv, not pip/apt).
3. **Non-root**: Runs as the `hermesagent` system user with scoped sudo.
4. **Idempotent**: All tasks guarded by `stat` checks or `creates:` — safe to re-run.
5. **No conflicts**: Do NOT reinstall Node.js, Docker, Tailscale, Ollama, or touch UFW.

## Task Order

```
roles/hermesagent/tasks/main.yml:
  prereqs.yml   → Minimal deps (git, curl, ca-certificates, python3-venv, acl)
  user.yml      → Create hermesagent user, sudoers, SSH keys
  install.yml   → Run install.sh as hermesagent, verify binary at ~/.local/bin/hermes
  service.yml   → Deploy + enable systemd service (skipped in ci_test mode)
  tailscale.yml → Configure tailscale serve on port 8444 (skipped in ci_test mode)
```

## Critical Notes

### Binary path
After install: `~/.local/bin/hermes` (symlink → `~/.hermes/hermes-agent/venv/bin/hermes`).
The idempotency guard checks this path.

### Python managed by uv
The install script downloads and manages Python 3.11 via `uv`. Do not add Python install tasks.
`python3-venv` must be present on the host for uv to bootstrap.

### Node.js already present
Node.js 22.x is installed system-wide by openclaw-ansible. The installer PATH includes
`/usr/bin` so it detects and reuses it — do not add a nodejs task.

### No UFW changes
hermesagent port is localhost-only. Do NOT add UFW rules.

### Port
Default `hermes_port: 8000`. Override via `-e hermes_port=<n>` if the gateway uses a different port.
Confirm after first install: `hermes config | grep port` or inspect `~/.hermes/config.yaml`.

### Service run command
`ExecStart={{ hermes_bin }} gateway` — runs the hermes gateway in foreground for systemd.

## Code Style

- No `become_user` at the play level — use per-task `become_user` where needed.
- Use `creates:` or `stat` checks to make tasks idempotent.
- Always specify `become_user: "{{ hermes_user }}"` when running installer or hermes commands.
- Pass `HOME` and `HERMES_INSTALL_DIR` in `environment:` for all hermesagent user tasks.

## Testing Checklist

```bash
# 1. Syntax check
ansible-playbook playbook.yml --syntax-check

# 2. Lint
ansible-lint playbook.yml

# 3. Dry run
ansible-playbook -i inventory playbook.yml --check --ask-become-pass

# 4. Full install (on target)
ansible-playbook -i inventory playbook.yml --ask-become-pass

# 5. Verify service
systemctl status hermesagent
curl http://127.0.0.1:8000/health
journalctl -u hermesagent -n 50

# 6. Confirm no port conflicts
ss -tlnp | grep -E '3000|3100|11434|8000'

# 7. Run Docker test suite (CI)
bash tests/run-tests.sh
```

## File Locations

### Host System
```
/home/hermesagent/.local/bin/hermes           # installed symlink
/home/hermesagent/.hermes/hermes-agent/       # app source + venv
/home/hermesagent/.hermes/config.yaml         # hermes config
/home/hermesagent/.hermes/.env                # API keys (on host, not in this repo)
/etc/systemd/system/hermesagent.service
/etc/sudoers.d/hermesagent
```

### Repository
```
roles/hermesagent/
├── tasks/
│   ├── main.yml
│   ├── prereqs.yml
│   ├── user.yml
│   ├── install.yml
│   ├── service.yml
│   └── tailscale.yml
├── templates/
│   └── hermesagent.service.j2
├── defaults/
│   └── main.yml
└── handlers/
    └── main.yml
```

## Making Changes

### Updating hermes-agent
Re-run install.yml tasks — the `creates:` guard means it only re-installs if the binary is absent.
To force reinstall: `rm /home/hermesagent/.local/bin/hermes` then re-run the playbook.

### Changing the Port
Override `hermes_port`. Updates the systemd service template and Tailscale serve config.

### Adding API Keys
Set `hermes_api_key` via `-e` or host_vars. The service template adds `HERMES_API_KEY` to the
systemd environment. Never commit key values to this repo.
