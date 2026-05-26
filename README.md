# VPS Quadlet Deployment

Ansible setup for deploying rootless Podman Quadlet services. All services run under an unprivileged `apprunner` user via systemd user units. Supports a production VPS and a local test VM from the same
codebase.

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
├── inventory.ini              # host aliases
├── requirements.yml
├── group_vars/all/
│   └── vars.yml               # shared config
├── host_vars/
│   ├── prod/
│   │   ├── vars.yml            # prod inputs (gitignored)
│   │   ├── vars.yml.example    # placeholder template
│   │   ├── vault.yml           # prod secrets, encrypted (gitignored)
│   │   └── vault.yml.example   # placeholder template
│   └── test/
│       ├── vars.yml            # test inputs
│       └── vault.yml           # test secrets, encrypted
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

2. Create the production host vars from the examples:

   ```sh
   cp host_vars/prod/vars.yml.example  host_vars/prod/vars.yml
   cp host_vars/prod/vault.yml.example host_vars/prod/vault.yml
   ```

3. Fill in real values, then encrypt the vault:

   ```sh
   ansible-vault encrypt host_vars/prod/vault.yml
   ```

   Edit an encrypted vault later with:

   ```sh
   ansible-vault edit host_vars/prod/vault.yml
   ansible-vault edit host_vars/test/vault.yml
   ```

## Usage

Run the one-time bootstrap (only needed again on system-level changes):

```sh
ansible-playbook bootstrap.yml --limit test --ask-vault-pass --ask-pass --ask-become-pass
ansible-playbook bootstrap.yml --limit prod --ask-vault-pass --ask-become-pass
```

Deploy all services to a host:

```sh
ansible-playbook deploy.yml --limit test --ask-vault-pass --ask-pass --ask-become-pass
ansible-playbook deploy.yml --limit prod --ask-vault-pass --ask-become-pass
```

Deploy a single service via its tag:

```sh
ansible-playbook deploy.yml --limit prod --tags caddy --ask-vault-pass --ask-become-pass
```

## Notes

- `inventory.ini` holds only SSH config aliases — no IPs or usernames.
- `host_vars/test/` is committed. `host_vars/prod/vars.yml` and `host_vars/prod/vault.yml`
  are gitignored; only their `.example` templates are committed.
- Quadlet files (`.container`, `.volume`, `.network`) are static. Environment
  values live in per-service `.env.j2` templates; secrets go through Ansible
  Vault and Podman secrets, never in plaintext.
- Test on the local VM first, then deploy to production with `--limit prod`.