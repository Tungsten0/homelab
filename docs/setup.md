# Setup and Secrets

This repository contains configuration, not production credentials or application data. A fresh deployment requires local secret files and a few environment-specific values before the Compose stacks can be started.

## Prerequisites

For the Docker host:

- Debian/Linux host with Docker Engine installed;
- Docker Compose v2 (`docker compose`);
- persistent storage available for Docker volumes/application data;
- connectivity to the Proxmox host and any monitored hosts;
- static/reserved LAN addressing, or equivalent DNS names, for Prometheus targets.

The checked-in Prometheus configuration currently expects:

```text
Proxmox:   192.168.2.10
fileserver: 192.168.2.127
docker01:   192.168.2.131
```

Change `stacks/monitoring/prometheus/prometheus.yml` and the exporter bind addresses if your environment uses different addresses.

## Repository Secret Policy

Real credentials must **never** be committed.

The root `.gitignore` excludes common secret/key files and secret directories. Individual stacks also ignore their local secret paths.

Recommended permissions when creating secret files:

```bash
umask 077
```

This causes newly created files to be readable/writable only by the creating user unless permissions are changed later.

## Nextcloud Secrets

Create the local secret directory:

```bash
cd stacks/nextcloud
mkdir -p secrets
chmod 700 secrets
```

Create the five required files:

```bash
printf '%s' 'nextcloud' > secrets/postgres_db
printf '%s' 'nextcloud' > secrets/postgres_user
printf '%s' 'REPLACE_WITH_A_RANDOM_DATABASE_PASSWORD' > secrets/postgres_password
printf '%s' 'admin' > secrets/nextcloud_admin_user
printf '%s' 'REPLACE_WITH_A_RANDOM_ADMIN_PASSWORD' > secrets/nextcloud_admin_password
chmod 600 secrets/*
```

The values above are examples only. Use unique random production passwords.

The Compose file passes the values through Docker secrets rather than embedding credentials directly in environment variables or source-controlled YAML.

### Start Nextcloud

The custom image installs SMB support for Nextcloud external storage.

```bash
cd stacks/nextcloud
docker compose build
docker compose up -d
```

Inspect status:

```bash
docker compose ps
docker compose logs --tail=100
```

## Monitoring Secrets

Create the monitoring secret directory:

```bash
cd stacks/monitoring
mkdir -p secrets
chmod 700 secrets
```

### Grafana Administrator Password

```bash
printf '%s' 'REPLACE_WITH_A_RANDOM_GRAFANA_PASSWORD' > secrets/grafana_admin_password
chmod 600 secrets/grafana_admin_password
```

The Grafana Compose service reads the password from `/run/secrets/grafana_admin_password`.

### Proxmox VE Exporter

Copy the safe template:

```bash
cp pve.yml.example secrets/pve.yml
chmod 600 secrets/pve.yml
```

Then replace all placeholder values in `secrets/pve.yml`.

The intended authentication model is a dedicated Proxmox metrics account/API token with read-only permissions. Do not use a root account or a general-purpose administrative API token for metrics collection.

Example structure:

```yaml
default:
  user: prometheus@pve
  token_name: "metrics"
  token_value: "REPLACE_WITH_REAL_TOKEN_VALUE"
  verify_ssl: true
```

If the Proxmox API uses a certificate that is not trusted by the exporter container, either add the appropriate CA trust or deliberately set `verify_ssl: false` after understanding the tradeoff. Keeping certificate verification enabled is preferred when practical.

### Start Monitoring

```bash
cd stacks/monitoring
docker compose pull
docker compose up -d
```

Check Prometheus targets and Grafana after startup. Prometheus should be able to reach:

- its own metrics endpoint;
- node-exporter instances;
- cAdvisor on `docker01`;
- the internal `pve-exporter` service.

## Docker Host Exporters

The exporter stack is separate from the main monitoring stack:

```bash
cd stacks/exporters/docker01
docker compose pull
docker compose up -d
```

The checked-in configuration binds exporter listeners specifically to `192.168.2.131`, the current `docker01` LAN address.

If `docker01` has another address, change both:

```yaml
--web.listen-address=<DOCKER01_LAN_IP>:9100
```

and:

```yaml
ports:
  - "<DOCKER01_LAN_IP>:8080:8080"
```

Then update Prometheus targets accordingly.

## Uptime Kuma

Uptime Kuma currently stores persistent data in:

```text
/srv/docker/appdata/uptime-kuma
```

Create/verify the parent storage path before deployment, then run:

```bash
cd stacks/uptime-kuma
docker compose pull
docker compose up -d
```

The Compose file currently specifies public DNS resolvers. If you monitor internal DNS names such as `*.home.arpa`, use the LAN resolver instead or remove the explicit `dns:` override so the host's normal resolver configuration can be used.

## Deployment Order

A practical first deployment order is:

1. Docker host exporters;
2. monitoring stack;
3. Nextcloud;
4. Uptime Kuma.

There is no strict dependency between all stacks, but bringing exporters up before Prometheus avoids unnecessary initial target failures.

## Pre-Publication Secret Check

Before making a Git repository public, inspect more than the current files. Scan the complete Git history for accidentally committed credentials.

At minimum, search for:

- `.env` files;
- private keys and certificates;
- Tailscale auth keys;
- Cloudflare tokens;
- Proxmox API token values;
- database passwords;
- Grafana/Nextcloud credentials.

If a real credential was ever committed, rotate/revoke it even after cleaning the repository history.
