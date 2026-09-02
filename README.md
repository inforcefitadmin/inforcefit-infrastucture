# InforceFit Infrastructure

Ansible configuration for the initial provisioning of the VPS that runs the InforceFit application. Application code deployment (prod/dev) is handled separately via GitHub Actions (`release.yml` in the `inforcefit-server` repository) — this repository is responsible **only for server provisioning**, not for shipping releases.
## Cold-starting the application (first run on a new server)

After `setup-vps.yml` has provisioned the server, run `bootstrap-app.yml` **once** to bring both prod and dev online from scratch: clones both repositories, pulls `.env` from Infisical, installs dependencies, builds, and starts both apps under PM2.

### Additional variables required

Add to `ansible/vars/main.yml`:
```yaml
infisical_domain: "https://app.infisical.com"
infisical_project_id: "YOUR_PROJECT_ID"
```

`infisical_client_id` and `infisical_client_secret` must **not** be committed in plain text. Either pass them at runtime:
```bash
ansible-playbook -i inventory/prod playbooks/bootstrap-app.yml \
  --extra-vars "infisical_client_id=xxx infisical_client_secret=yyy"
```

or store them in an Ansible Vault file:
```bash
ansible-vault create ansible/vars/secrets.yml
```
```yaml
infisical_client_id: "xxx"
infisical_client_secret: "yyy"
```
and reference it in `bootstrap-app.yml`'s `vars_files`.

### Running

```bash
cd ansible
ansible-playbook -i inventory/prod playbooks/bootstrap-app.yml --ask-vault-pass
```

This step is **idempotent for the PM2 start**: if a process with the same name is already running, it will be skipped rather than duplicated. It is **not** idempotent for cloning — re-running against an already-cloned directory will simply skip the clone step (`update: no`), so subsequent code updates must go through the normal GitHub Actions deploy pipeline, not this playbook.

Run it once per fresh server (or per environment, if prod/dev live on separate hosts). Ongoing deployments after this point are handled exclusively by `release.yml` in GitHub Actions — this playbook is only for the initial cold start.

## What this playbook does

Running `setup-vps.yml` executes the following roles in order:

1. **base** — base packages, `deploy` user, UFW firewall, fail2ban, automatic security updates
2. **security** — auditd with rules for monitoring critical directories (SSH keys, cron, systemd units, `.env` files, nginx config)
3. **nodejs** — Node.js 20.x, PM2 + pnpm, PM2 auto-start via systemd, PM2 log rotation
4. **nginx** — nginx + Certbot, reverse proxy configuration for prod/dev
5. **monitoring** — Grafana + Prometheus + Loki (native installation)

## Prerequisites

- Fresh Ubuntu 24.04 VPS (minimum 2 vCPU / 4GB RAM — less is not recommended due to Grafana+Prometheus+Loki)
- SSH access with sudo privileges
- Ansible ≥ 2.15 installed on the local machine
- Public SSH key added to the server (for Ansible to connect)

## Structure

ansible/
├── inventory/
│ ├── prod # IP/hostname of the production server
│ └── dev # IP/hostname of the dev server (if separate) or the same host
├── vars/
│ └── main.yml # all variables: domains, ports, PM2 process names
├── playbooks/
│ └── setup-vps.yml # main playbook for initial provisioning
└── roles/
├── base/
├── security/
├── nodejs/
├── nginx/
└── monitoring/


## Setup before running

1. Edit `ansible/inventory/prod` (and `dev`, if it's a different server):
```ini
   [webservers]
   your-server-ip ansible_user=root
```

2. Review and adjust `ansible/vars/main.yml`:
   - `prod_domain` / `dev_domain` — your actual domains
   - `ssh_port` — if SSH is not on the standard port 22
   - `app_user` — the user PM2 runs as (defaults to `deploy`)

3. Review the nginx template (`ansible/roles/nginx/templates/inforcefit.conf.j2`) — make sure `proxy_pass` points to the correct ports (`prod_app_port: 3000`, `dev_app_port: 3001`).

## Running

Initial setup of a new production server:
```bash
cd ansible
ansible-playbook -i inventory/prod playbooks/setup-vps.yml
```

Same for the dev server (if it's a separate host):
```bash
ansible-playbook -i inventory/dev playbooks/setup-vps.yml
```

⚠️ **Before running this on a real production server, test the full playbook on a clean test VPS first.** The `monitoring` role installs Grafana/Prometheus/Loki natively via apt repositories — if the target server already has these services installed some other way, review the configuration manually first to avoid port conflicts or duplicate systemd units.

## Application deployment (prod/dev)

Once `setup-vps.yml` has prepared the server (Node.js, PM2, nginx, security), **the application code itself is deployed via GitHub Actions**, not via Ansible:

- Push to `dev` → automatic deployment to the dev environment
- Manual `workflow_dispatch` run with confirmation → deployment to `master` (production)

The deployment pipeline (`.github/workflows/release.yml` in the `inforcefit-server` repository) handles:
- Pulling secrets from Infisical → `.env`
- `pnpm install --frozen-lockfile` + `pnpm run build`
- Zero-downtime `pm2 reload`
- Health check with automatic rollback on failure

This is a deliberate separation of concerns: Ansible = server infrastructure, GitHub Actions = code lifecycle.

## Verification after provisioning

After a successful `setup-vps.yml` run:

```bash
# Check that all services are active
ssh deploy@your-server-ip 'systemctl status nginx pm2-deploy auditd fail2ban grafana-server prometheus'

# Check UFW
ssh deploy@your-server-ip 'sudo ufw status'

# Check auditd rules
ssh deploy@your-server-ip 'sudo auditctl -l'
```

After that, the first code deployment via GitHub Actions (push to `dev`, or a manual run on `master`) will bring the application up under PM2.

## Known limitations / TODO

- Loki retention configuration (deleting old logs) is not automated in this role yet — it's set up manually on the server after installation.
- The `monitoring` role is designed for a **new** server; for a server that already has a monitoring stack set up manually, a manual review is needed before applying it.
- SSL certificates via Certbot require a manual `certbot --nginx -d your-domain` run after nginx is installed (not automated in this role).
- `bootstrap-app.yml` is meant for first-time cold start only; it does not handle subsequent code updates, migrations, or rollbacks — those remain the responsibility of `release.yml` (GitHub Actions).
