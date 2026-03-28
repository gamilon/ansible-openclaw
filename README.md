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
| `site.yml` | Main play: `common` then `openclaw`. |
| `group_vars/all.yml` | Example: `ansible_user` for SSH and authorized key placement. |
| `files/id_rsa.pub` | Public key installed by the `common` role (create locally; `files/` is gitignored). |
| `roles/common/` | Baseline OS packages, chrony, SSH hardening, authorized key. |
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

Roles are tagged **`common`** and **`openclaw`**. Finer tags let you run subsets:

| Tag | Role | What runs |
|-----|------|-----------|
| `apt` | common | apt cache refresh |
| `packages` | common | apt upgrade + install `common_packages` |
| `chrony` | common | chrony service |
| `ssh` | common | authorized key + sshd hardening |
| `debug` | common | completion debug task |
| `vault` | common | HashiCorp Vault CLI zip from releases.hashicorp.com (needs `common_install_vault_cli: true`) |
| `nodejs` | openclaw | NodeSource + Node.js |
| `users` | openclaw | `openclaw` system user |
| `docker` | openclaw | Docker packages + `docker` group |
| `gateway` | openclaw | npm install + systemd unit |

The Ubuntu-only check in `common` is tagged **`always`** so it still runs when you use `--tags` (unless you `--skip-tags always`).

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
- **Vault CLI** (optional): if `common_install_vault_cli` is `true`, downloads `vault_{{ common_vault_version }}_linux_{amd64|arm64}.zip` from [releases.hashicorp.com](https://releases.hashicorp.com/vault/) and installs the binary to `common_vault_install_dir` (default `/usr/local/bin`).

### Role: `openclaw`

- **Node.js** via [NodeSource](https://github.com/nodesource/distributions) (`openclaw_nodejs_major_version`, default `22`).
- System user **`openclaw`** with home **`/opt/openclaw`**.
- **npm** global install of the `openclaw` package under **`openclaw_npm_prefix`** (defaults to `/opt/openclaw` → CLI at `/opt/openclaw/bin/openclaw`).
- **systemd** unit **`openclaw-gateway.service`**: runs `openclaw gateway run` as `openclaw`, with `OPENCLAW_NO_RESPAWN=1`.
- **Docker** (optional): `docker.io` + `docker-compose-v2`; users in `openclaw_docker_users` are added to the `docker` group (includes `ansible_user` and `openclaw` by default).

## Configuration

**Common role** (`roles/common/defaults/main.yml`):

- `common_install_vault_cli` — set `true` to install the Vault CLI from HashiCorp releases (default `false`).
- `common_vault_version` — release version string, e.g. `1.18.5` (must match a published zip on releases.hashicorp.com).
- `common_vault_install_dir` — directory for the `vault` binary (default `/usr/local/bin`).
- Zip suffix (`linux_amd64` vs `linux_arm64`) is derived from the gathered fact **`ansible_architecture`** (`x86_64` / `aarch64`).

**OpenClaw role** — most knobs live in **`roles/openclaw/defaults/main.yml`**. Override them in `group_vars/` or `host_vars/` as needed, for example:

- `openclaw_install_docker_for_agents` — set `false` if this host does not run Docker-based agents.
- `openclaw_gateway_extra_args` — extra flags for `openclaw gateway run` (e.g. port).
- `openclaw_gateway_allow_unconfigured` — dev-only; adds `--allow-unconfigured`. For production, configure `gateway.mode` and auth under `/opt/openclaw/.openclaw/` (or run onboarding as the `openclaw` user).
- `openclaw_gateway_service_environment` — list of `KEY=value` strings for extra `Environment=` lines in the unit.
- `openclaw_npm_install_force` — set `true` to always run `npm install` for the gateway CLI (default skips when `openclaw` already exists under `openclaw_npm_version: latest`, or when a pinned version already matches `package.json`).

## After the first run

- **OpenClaw config:** The gateway expects normal OpenClaw configuration (see upstream docs). Until that exists, the service may fail to stay up unless you use `openclaw_gateway_allow_unconfigured` for testing.
- **Docker group:** Users added to `docker` need a **new login session** (or `newgrp docker`) before `docker` works without sudo.
- **Onboarding:** Interactive steps (e.g. `openclaw onboard`) are not automated here; run them as appropriate for your environment, typically as the `openclaw` user with `sudo -u openclaw -H bash` from the host.

## Security notes

- Do **not** commit **private** keys. The `files/` directory is gitignored; keep keys only on your control machine or supply them via CI secrets.
- The `common` role disables password SSH; ensure key access works before applying to production hosts.
- Review `roles/common/tasks/main.yml` SSH block if you need `AllowUsers`, non-default ports, or other policy.

## License

This project is licensed under the [MIT License](LICENSE).