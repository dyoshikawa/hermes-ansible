# hermes-ansible

Ansible playbook that installs [Hermes Agent](https://github.com/NousResearch/hermes-agent) on an
Ubuntu LTS VPS and runs its messaging gateway as a systemd service that comes back after a reboot.

What the `hermes_agent` role does:

1. Installs the packages the official installer needs (`git`, `curl`, `xz-utils`, `ripgrep`, `acl`).
2. Creates an unprivileged service account (`hermes`) with a private home directory.
3. Downloads the official installer from the pinned hermes-agent commit, verifies its SHA-256,
   and runs it as that account with `--skip-setup --commit <sha>` and, for a headless server,
   `--skip-browser --skip-computer-use`. The installer brings its own uv, Python and Node.js.
4. Writes your API keys and bot tokens to `~hermes/.hermes/.env` (mode 0600) and merges your
   settings into `~hermes/.hermes/config.yaml`.
5. Optionally pre-installs Python extras such as `messaging` (Telegram / Discord / Slack).
6. Installs the gateway as a boot-time **system** service running as `hermes`, sandboxed by a
   systemd hardening drop-in, and restarts it whenever `.env` or `config.yaml` change.
7. Installs a root-owned `/usr/local/bin/hermes` wrapper that always runs Hermes as `hermes`.

Supported targets: Ubuntu 24.04 LTS and 26.04 LTS on x86-64 or arm64. CI provisions both releases
from scratch on every push and checks that a second run makes no changes.

## Requirements

On the machine you run Ansible from:

- [mise](https://mise.jdx.dev/) — installs Python, [uv](https://docs.astral.sh/uv/) and the linters
  from `mise.toml`. Without mise, any Python ≥ 3.12 with uv works.
- SSH access to the VPS as a user with passwordless `sudo` (the playbook uses `become`).

On the VPS: a fresh Ubuntu LTS with outbound internet access. Nothing else — the role installs what it
needs.

## Quick start

```bash
git clone https://github.com/dyoshikawa/hermes-ansible.git
cd hermes-ansible
mise install          # Python, uv, actionlint, pinact
mise run install      # uv sync: ansible-core + linters into .venv

# 1. Inventory
cp inventory/hosts.example.yml inventory/hosts.yml
$EDITOR inventory/hosts.yml           # ansible_host / ansible_user of your VPS

# 2. Secrets (git-ignored, encrypted with Ansible Vault)
cp inventory/group_vars/hermes/vault.example.yml inventory/group_vars/hermes/vault.yml
$EDITOR inventory/group_vars/hermes/vault.yml
uv run ansible-vault encrypt inventory/group_vars/hermes/vault.yml

# 3. Non-secret settings (model, provider, …)
$EDITOR inventory/group_vars/hermes/vars.yml

# 4. Provision
uv run ansible-playbook site.yml --ask-vault-pass
```

Afterwards, on the VPS:

```bash
sudo systemctl status hermes-gateway
sudo -u hermes -H hermes doctor
sudo -u hermes -H hermes            # interactive chat as the service account
```

Re-running the playbook is safe and is also how you roll out changed settings — the gateway restarts
only when `.env`, `config.yaml` or the code changed.

## Configuration

All variables live in `roles/hermes_agent/defaults/main.yml`; the most useful ones:

| Variable | Default | Purpose |
| --- | --- | --- |
| `hermes_agent_user` | `hermes` | Service account that owns the install and runs the gateway. |
| `hermes_agent_env` | `{}` | Key/value pairs written to `~/.hermes/.env` — API keys, bot tokens, allow-lists. Point the values at Vault variables. |
| `hermes_agent_env_exclusive` | `false` | `true` replaces `.env` with exactly `hermes_agent_env`; `false` merges keys into the file Hermes seeded. |
| `hermes_agent_config` | `{}` | Nested settings merged into `~/.hermes/config.yaml` (e.g. `model.provider`, `model.default`). |
| `hermes_agent_extras` | `[]` | Extras from Hermes' `pyproject.toml` to pre-install, e.g. `["messaging"]`. Hermes lazy-installs them on first use otherwise. |
| `hermes_agent_gateway_enabled` | `true` | Install and run the gateway systemd service. |
| `hermes_agent_gateway_state` | `started` | `started` or `stopped`. |
| `hermes_agent_branch` / `hermes_agent_commit` | `main` / v0.22.2's SHA | Branch and commit of hermes-agent to install. A changed commit is rolled out on the next play. |
| `hermes_agent_installer_url` / `hermes_agent_installer_checksum` | `scripts/install.sh` at that commit / its `sha256:` | Installer and the checksum it must match before it runs. |
| `hermes_agent_installer_allow_unverified` | `false` | Allow an empty checksum (e.g. to follow a branch tip with `hermes_agent_commit: ""`). |
| `hermes_agent_update` | `false` | Re-run the installer on every play (repairs a pinned install; pulls the tip when unpinned). |
| `hermes_agent_gateway_hardening_enabled` / `hermes_agent_gateway_hardening` | `true` / see defaults | systemd sandboxing drop-in for the gateway (`NoNewPrivileges`, `ProtectSystem=strict`, `PrivateTmp`, …). |
| `hermes_agent_gateway_hardening_extra` | `{}` | Directives merged on top of `hermes_agent_gateway_hardening`, e.g. `{ReadWritePaths: "/home/hermes /srv/data"}`. Overriding `hermes_agent_gateway_hardening` itself replaces the whole set. |
| `hermes_agent_system_command` | `true` | Install the root-owned `/usr/local/bin/hermes` wrapper (`false` removes it). |
| `hermes_agent_install_browser` | `false` | Keep Playwright/Chromium (browser tools). Off for headless servers. |
| `hermes_agent_extra_apt_packages` | `[]` | Extra apt packages, e.g. `ffmpeg` for voice notes. |
| `hermes_agent_user_groups` | `[]` | Supplementary groups, e.g. `["docker"]` for `terminal.backend: docker` — see [Security notes](#security-notes). |
| `hermes_agent_supported_releases` | `["24.04", "26.04"]` | Ubuntu releases the preflight accepts; `hermes_agent_check_distribution: false` disables the check. |

### Example: Telegram bot on OpenRouter

`inventory/group_vars/hermes/vars.yml`

```yaml
hermes_agent_config:
  model:
    provider: openrouter
    default: anthropic/claude-opus-5

hermes_agent_extras: [messaging]

hermes_agent_env:
  OPENROUTER_API_KEY: "{{ vault_openrouter_api_key }}"
  TELEGRAM_BOT_TOKEN: "{{ vault_telegram_bot_token }}"
  TELEGRAM_ALLOWED_USERS: "{{ vault_telegram_allowed_users }}"
```

`inventory/group_vars/hermes/vault.yml` (encrypted)

```yaml
vault_openrouter_api_key: sk-or-…
vault_telegram_bot_token: 123456789:ABC…
vault_telegram_allowed_users: "123456789"
```

The env var names are the ones Hermes documents for each platform — see the
[messaging guide](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/).

### Updating Hermes

The role installs a pinned commit (a full 40-character SHA). To move to a newer one, set
`hermes_agent_commit` and the checksum of the installer at that commit, then re-run the playbook:

```bash
sha=<new commit>
curl -fsSL "https://raw.githubusercontent.com/NousResearch/hermes-agent/$sha/scripts/install.sh" | sha256sum
```

```yaml
hermes_agent_commit: <new commit>
hermes_agent_installer_checksum: "sha256:<output above>"
```

To track the branch tip instead (not reproducible, and the installer is not verified), set
`hermes_agent_commit: ""`, `hermes_agent_installer_checksum: ""`,
`hermes_agent_installer_allow_unverified: true` and run with `-e hermes_agent_update=true`.

Avoid `hermes update` on the VPS while a commit is pinned: it moves the checkout off the pin, and
the next play re-runs the installer to go back to it, which the installer refuses as a rollback
unless `hermes_agent_force_commit: true`.

Manage the service with `sudo systemctl … hermes-gateway`. `hermes gateway install|start|restart
--system` need root, and the wrapper always runs Hermes as `hermes`, so they fail through it.

## Security notes

- Root never executes code from the checkout the `hermes` account can write to, and never reads or
  writes files under `~hermes` (those tasks run as `hermes`, so planted symlinks gain nothing). The
  gateway unit is generated by Hermes' own code running as `hermes`; root only installs it after
  every line matches the exact shape Hermes emits — `User=hermes`, fixed values for output,
  dependencies and `Exec` prefixes, plain path characters elsewhere. `/usr/local/bin/hermes` is a root-owned wrapper that switches to `hermes` (via `runuser` for
  root, `sudo -u hermes` for other admins).
- The gateway runs with a hardening drop-in
  (`/etc/systemd/system/hermes-gateway.service.d/50-ansible-hardening.conf`): no new privileges, no
  capabilities, a private `/tmp` and a read-only filesystem except the `hermes` home. If a tool you
  enable needs to write elsewhere, set `ReadWritePaths` in `hermes_agent_gateway_hardening_extra`.
  The sandbox covers the gateway process tree only: cron and Kanban workers that Hermes starts through
  `systemd-run --user` run under `user@<uid>.service` without it, so treat it as defence in depth,
  not as a boundary the agent cannot cross.
- `hermes_agent_installer_checksum` pins `install.sh` itself. What the installer downloads next
  (uv, Python, Node.js, Python wheels) is verified by the installer's own pinned hashes and
  `uv.lock`, not by this role.
- Membership in the `docker` group is **root-equivalent**: anyone in it can start a privileged
  container that mounts `/`. Adding `docker` to `hermes_agent_user_groups` therefore gives the agent
  (and anything that compromises it) root on the VPS. Prefer rootless Docker or a dedicated VM when
  you need the docker terminal backend.

## Layout

```
site.yml                      Playbook: applies hermes_agent to the `hermes` group
inventory/hosts.example.yml   Copy to hosts.yml (git-ignored)
inventory/group_vars/hermes/  vars.yml (tracked) + vault.yml (git-ignored, Vault-encrypted)
roles/hermes_agent/           The role: preflight → system → install → configure → gateway
tests/inventory.yml           localhost inventory used by CI to provision the runner itself
```

## Development

```bash
mise run check    # yamllint, ansible-lint, syntax check, gitleaks, actionlint, pinact
mise run lint
mise run syntax
```

CI (`.github/workflows/ci.yml`) lints, scans the history with gitleaks (the repository is public), then runs the playbook twice against `ubuntu-24.04` and
`ubuntu-26.04` runners, asserting the second run reports `changed=0`, that `hermes --version` works
for the service account and through the root-owned wrapper, that the checkout sits at the pinned
commit, that `.env` / `config.yaml` contain the expected values and that `hermes-gateway` is
enabled, active, sandboxed and not restarting.

## License

MIT
