# ansible-openclaw

Ansible playbooks to prepare **Ubuntu** hosts for a **hybrid OpenClaw** layout:

- **Gateway** runs on the host under **systemd**, as a dedicated system user, with **Node.js** and a global **npm** install under `/opt/openclaw`.
- **Agents** are intended to run in **Docker** containers on the same host (Docker Engine + Compose plugin are installed when enabled).

Upstream docs: [OpenClaw installation](https://openclawdoc.com/docs/getting-started/installation/), [Gateway CLI](https://openclaws.io/docs/cli/gateway).

## Requirements

- **Ansible** 2.14+ (uses `ansible.builtin` and standard patterns).
- **Target hosts:** Ubuntu only (the `common` role fails on other distributions).
- **SSH:** Your admin user must be able to connect with privilege escalation (`become`). The `common` role installs your SSH public key and hardens `sshd` to key-only auth.

## Layout

| Path | Purpose |
|------|---------|
| `site.yml` | Main play: `common`, optional `vault_client`, then `openclaw`. |
| `group_vars/all.yml` | Example: `ansible_user` for SSH and authorized key placement. |
| `files/id_rsa.pub` | Public key installed by the `common` role (create locally; `files/` is gitignored). |
| `roles/common/` | Baseline OS packages, chrony, SSH hardening, authorized key. |
| `roles/vault_client/` | Optional HashiCorp Vault CLI from releases, optional CA + `profile.d` (`VAULT_ADDR` / `VAULT_CACERT`). |
| `roles/openclaw/` | NodeSource Node.js, Docker (optional), `openclaw` system user, npm install, `openclaw-gateway` systemd unit. |

## Quick start

1. **Inventory** — create an inventory file (example `inventory.ini`):

   ```ini
   [openclaw_hosts]
   myserver.example.com ansible_host=203.0.113.10
   ```

2. **SSH user** — set `ansible_user` in `group_vars/all.yml` or per-host so it matches an account that already exists on the server and may use `sudo`.

3. **Public key** — place the matching **public** key at `files/id_rsa.pub` on the machine that runs Ansible (not committed; `files/` is ignored).

4. **Run the play:**

   ```bash
   ansible-playbook -i inventory.ini site.yml
   ```

## Tags

Roles are tagged **`common`**, **`vault_client`**, and **`openclaw`**. Finer tags let you run subsets:

| Tag | Role | What runs |
|-----|------|-----------|
| `apt` | common | apt cache refresh |
| `packages` | common | apt upgrade + install `common_packages` |
| `chrony` | common | chrony service |
| `ssh` | common | authorized key + sshd hardening |
| `debug` | common | completion debug task |
| `vault_client` | vault_client | Whole role (only if `vault_client_install_cli` is `true`) |
| `vault` | vault_client | Vault CLI zip install + version display |
| `vault_profile` | vault_client | CA copy + `/etc/profile.d` snippet (`VAULT_ADDR` / `VAULT_CACERT`); role must be enabled (`vault_client_install_cli`) |
| `nodejs` | openclaw | NodeSource + Node.js |
| `users` | openclaw | `openclaw` system user |
| `docker` | openclaw | Docker packages + `docker` group |
| `gateway` | openclaw | npm install, optional non-interactive onboard, systemd unit |
| `onboard` | openclaw | `openclaw onboard --non-interactive` only (when `openclaw_onboard_enabled`; also runs under `gateway` tag) |

The Ubuntu-only checks in `common` and `vault_client` are tagged **`always`** so they still run when you use `--tags` (unless you `--skip-tags always`). If `vault_client_install_cli` is `false`, the `vault_client` role is skipped (including when using `--tags vault`).

Examples:

```bash
ansible-playbook -i inventory site.yml --tags ssh
ansible-playbook -i inventory site.yml --tags "gateway,docker"
ansible-playbook -i inventory site.yml --skip-tags debug
```

## What gets installed

### Role: `common`

- Ubuntu-only gate; apt update and package upgrade (`upgrade: yes`, not full dist-upgrade).
- Baseline packages (see `roles/common/defaults/main.yml`).
- **chrony** (default distro config).
- Your key in `~/.ssh/authorized_keys` for `{{ ansible_user | default('ubuntu') }}`.
- SSH hardening block: key-based auth only, `PermitRootLogin no`.

### Role: `vault_client` (optional)

Included from `site.yml` only when **`vault_client_install_cli`** is `true` (default `false` in role defaults). When enabled, installs the CLI from [releases.hashicorp.com](https://releases.hashicorp.com/vault/), optionally copies **`vault_client_ca_src`** (default `files/vault_ca.pem` on the controller) to **`vault_client_cacert_remote_path`**, and manages **`vault_client_profile_d_path`** with `VAULT_ADDR` and/or `VAULT_CACERT` when configured (see `roles/vault_client/defaults/main.yml`).

### Role: `openclaw`

- **Node.js** via [NodeSource](https://github.com/nodesource/distributions) (`openclaw_nodejs_major_version`, default `22`).
- System user **`openclaw`** with home **`/opt/openclaw`**.
- **npm** global install of the `openclaw` package under **`openclaw_npm_prefix`** (defaults to `/opt/openclaw` → real binary at `/opt/openclaw/bin/openclaw`). A symlink **`openclaw_cli_symlink`** (default `/usr/local/bin/openclaw`) is created so `openclaw` is on the normal PATH for root and other users (disable with **`openclaw_link_cli_in_path: false`**).
- **systemd** unit **`openclaw-gateway.service`**: runs `openclaw gateway run` as `openclaw`, with `OPENCLAW_NO_RESPAWN=1`.
- **Onboarding** (optional): when **`openclaw_onboard_enabled`** is `true`, runs **`openclaw onboard --non-interactive`** as the `openclaw` user after the CLI install and before the unit is installed, so workspace config exists before the service starts. See [openclaw onboard](https://openclaws.io/docs/cli/onboard) and [CLI automation](https://openclaws.io/docs/start/wizard-cli-automation).
- **Docker** (optional): `docker.io` + `docker-compose-v2`; users in `openclaw_docker_users` are added to the `docker` group (includes `ansible_user` and `openclaw` by default).

## Configuration

**Vault client role** (`roles/vault_client/defaults/main.yml`):

- `vault_client_install_cli` — set `true` in `group_vars` / `host_vars` to run the role (default `false`; `site.yml` skips the role when false).
- `vault_client_version` — release version string, e.g. `1.18.5` (must match a published zip on releases.hashicorp.com).
- `vault_client_install_dir` — directory for the `vault` binary (default `/usr/local/bin`).
- Zip suffix (`linux_amd64` vs `linux_arm64`) is derived from the gathered fact **`ansible_architecture`** (`x86_64` / `aarch64`).
- `vault_client_addr` — Vault server URL (e.g. `https://vault.example.com:8200`). Non-empty adds **`VAULT_ADDR`** to profile.d.
- `vault_client_manage_ca` — set `true` to copy **`vault_client_ca_src`** to **`vault_client_cacert_remote_path`** and add **`VAULT_CACERT`** to profile.d.
- `vault_client_ca_src` — PEM on the Ansible controller (default `{{ playbook_dir }}/files/vault_ca.pem`).
- `vault_client_cacert_remote_path` — PEM path on the target (default `/etc/vault/vault_ca.pem`).
- `vault_client_profile_d_path` — profile.d script (default `/etc/profile.d/vault.sh`; keep the **`.sh`** suffix on Ubuntu). The file is removed when both `vault_client_addr` is empty and `vault_client_manage_ca` is `false`.

**OpenClaw role** — most knobs live in **`roles/openclaw/defaults/main.yml`**. Override them in `group_vars/` or `host_vars/` as needed, for example:

- `openclaw_install_docker_for_agents` — set `false` if this host does not run Docker-based agents.
- `openclaw_gateway_extra_args` — extra flags for `openclaw gateway run` (e.g. port).
- `openclaw_gateway_service_enabled` / `openclaw_gateway_service_state` — control systemd (`enabled` on boot, `state` e.g. `started` or `stopped`). With no real config yet, use `enabled: false` and `state: stopped` so the unit does not flap and Ansible does not keep forcing `started` on every run.
- `openclaw_gateway_allow_unconfigured` — dev-only; adds `--allow-unconfigured`. For production, configure `gateway.mode` and auth under `/opt/openclaw/.openclaw/` (or run onboarding as the `openclaw` user).
- `openclaw_gateway_service_environment` — list of `KEY=value` strings for extra `Environment=` lines in the unit.
- `openclaw_npm_install_force` — set `true` to always run `npm install` for the gateway CLI (default skips when `openclaw` already exists under `openclaw_npm_version: latest`, or when a pinned version already matches `package.json`).
- `openclaw_link_cli_in_path` / `openclaw_cli_symlink` — symlink the CLI into a directory on default `PATH` (defaults: `true`, `/usr/local/bin/openclaw`).

**OpenClaw onboard** (`roles/openclaw/defaults/main.yml` — [CLI reference](https://openclaws.io/docs/start/wizard-cli-reference)):

- `openclaw_onboard_enabled` — set `true` to run non-interactive onboarding (default `false`). Does **not** use `--install-daemon`; this repo keeps the gateway on **systemd**.
- `openclaw_onboard_workspace` — workspace directory (default `{{ openclaw_system_user_home }}/.openclaw`).
- `openclaw_onboard_skip_health` — default `true` so automation does not require a running local gateway before `onboard` exits.
- `openclaw_onboard_skip_skills`, `openclaw_onboard_accept_risk`, `openclaw_onboard_json` — map to matching CLI flags.
- `openclaw_onboard_extra_argv` — list of extra arguments after `openclaw onboard` (e.g. `--mode`, `local`, `--auth-choice`, `openai-api-key`, `--secret-input-mode`, `ref`). You must supply whatever your chosen auth flow requires.
- `openclaw_onboard_environment` — dict merged with `HOME` for the onboard process; use for API keys (`OPENAI_API_KEY`, etc.) in **`ref`** mode or gateway token env vars (see upstream `--gateway-token-ref-env` examples).
- `openclaw_onboard_creates` — optional path for Ansible `creates` idempotency (skip onboard when this file exists). Leave empty to run every play while enabled.
- `openclaw_onboard_no_log` — default `true` to avoid logging environment/invocation details when secrets are present.

Example (`group_vars` fragment, OpenAI ref mode — keys must exist in `openclaw_onboard_environment` or on the host):

```yaml
openclaw_onboard_enabled: true
openclaw_onboard_accept_risk: true
openclaw_onboard_extra_argv:
  - --mode
  - local
  - --auth-choice
  - openai-api-key
  - --secret-input-mode
  - ref
openclaw_onboard_environment:
  OPENAI_API_KEY: "{{ lookup('env', 'OPENAI_API_KEY') }}"  # or vault lookup, ansible-vault var, etc.
```

## After the first run

- **OpenClaw config:** With onboard disabled, configure the gateway manually (see upstream docs) or enable **`openclaw_onboard_enabled`** so `openclaw onboard` runs before the systemd unit is written. Until config exists, the service may fail to stay up unless you use `openclaw_gateway_allow_unconfigured` for testing.
- **Docker group:** Users added to `docker` need a **new login session** (or `newgrp docker`) before `docker` works without sudo.
- **Onboarding:** For ad hoc interactive setup, run `sudo -u openclaw -H bash` on the host and use `openclaw onboard` without `--non-interactive`.

## Security notes

- Do **not** commit **private** keys. The `files/` directory is gitignored; keep keys only on your control machine or supply them via CI secrets.
- The `common` role disables password SSH; ensure key access works before applying to production hosts.
- Review `roles/common/tasks/main.yml` SSH block if you need `AllowUsers`, non-default ports, or other policy.

## License

This project is licensed under the [MIT License](LICENSE).