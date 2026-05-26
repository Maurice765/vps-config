# VPS Quadlet Deployment

Ansible setup for deploying rootless [Podman Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
services on a single VPS. All services run under an unprivileged `apprunner`
user via systemd user units.

## How it works

- **`bootstrap.yml`** — one-time, root-level setup: creates the `apprunner`
  user, enables lingering, allows privileged ports, creates base directories.
- **`deploy.yml`** — day-to-day deployment: loops over every service and applies
  the generic `quadlet_service` role.
- **`roles/quadlet_service/`** — the deployment logic, written once. Copies
  Quadlet files, native systemd units and config, renders templates, creates
  Podman secrets, and restarts only what changed.
- **`services/<name>/`** — one folder per service, pure data: a `vars.yml`
  manifest, static input files in `files/`, and Jinja2 templates in `templates/`.

Adding a service means creating one folder under `services/` and adding its
name to the `services` list in `deploy.yml` — no role changes.

## Layout

```
.
├── bootstrap.yml              # one-time root setup
├── deploy.yml                 # deploy all services
├── ansible.cfg
├── inventory.ini              # real VPS address (gitignored)
├── inventory.ini.example      # placeholder template
├── requirements.yml
├── group_vars/all/
│   ├── vars.yml               # real config (gitignored)
│   ├── vars.yml.example       # placeholder template
│   ├── vault.yml              # encrypted secrets (gitignored)
│   └── vault.yml.example      # placeholder template
├── roles/quadlet_service/
└── services/<name>/
    ├── vars.yml               # file lists, start units, secrets
    ├── files/                 # Quadlet files, native units, static config
    └── templates/             # *.j2 rendered onto the host
```

## Setup

1. Install the required collections:

   ```sh
   ansible-galaxy collection install -r requirements.yml
   ```

2. Create your variable files from the examples:

   ```sh
   cp inventory.ini.example            inventory.ini
   cp group_vars/all/vars.yml.example  group_vars/all/vars.yml
   cp group_vars/all/vault.yml.example group_vars/all/vault.yml
   ```

   Fill in real values, then encrypt the vault:

   ```sh
   ansible-vault encrypt group_vars/all/vault.yml
   ```

3. Set the VPS address and SSH user in `inventory.ini`.

## Usage

Run the one-time bootstrap (only needed again on system-level changes):

```sh
ansible-playbook bootstrap.yml
```

Deploy all services:

```sh
ansible-playbook deploy.yml
```

Deploy a single service via its tag:

```sh
ansible-playbook deploy.yml --tags caddy
```

## Notes

- `inventory.ini`, `vars.yml` and `vault.yml` are gitignored; only the `.example` files are
  committed. The vault file is encrypted — keep a backup, since it is not in
  the repo.
- Quadlet files (`.container`, `.volume`, `.network`) are static. Environment
  values live in per-service `.env.j2` templates; secrets go through Ansible
  Vault and Podman secrets, never in plaintext.
