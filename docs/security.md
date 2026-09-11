# Security and Network Exposure

This homelab is intended to operate on a trusted private network with remote administrative access handled through Tailscale. Backend and infrastructure endpoints should not be exposed directly to the public Internet.

## Principles

The current configuration follows several basic rules:

1. **Credentials stay out of Git.** Secret values are provided through ignored local files/Docker secrets.
2. **Databases are internal.** PostgreSQL and Redis are not published to the host network.
3. **Infrastructure exporters receive minimal exposure.** Exporters are either kept inside Docker networking or bound only to the required LAN interface.
4. **Public ingress is separate from backend service ports.** Future Internet-facing access should use a reverse proxy/tunnel rather than router port-forwarding to arbitrary container ports.
5. **Remote administration uses a private overlay where practical.** Tailscale provides remote connectivity without making management endpoints directly public.

## Current Exposure Matrix

| Component | Listener / publication | Intended clients | Notes |
| --- | --- | --- | --- |
| Nextcloud | `0.0.0.0:8081 -> 80` | Trusted LAN; future ingress layer | User-facing application |
| Grafana | `0.0.0.0:3000 -> 3000` | Trusted administrators | Sign-up and anonymous access disabled |
| Prometheus | `0.0.0.0:9090 -> 9090` | Trusted administrators | Should not be public Internet-facing |
| Uptime Kuma | `0.0.0.0:3001 -> 3001` | Trusted administrators | Application auth still required |
| PostgreSQL | Docker network only | Nextcloud | No host port |
| Redis | Docker network only | Nextcloud | No host port |
| PVE exporter | Docker network only | Prometheus | No host-published port |
| node-exporter (`docker01`) | `192.168.2.131:9100` | Prometheus/trusted LAN | Bound only to LAN interface |
| cAdvisor (`docker01`) | `192.168.2.131:8080` | Prometheus/trusted LAN | Bound only to LAN interface |

`0.0.0.0` above means Docker publishes on all host interfaces. These application/admin ports therefore still depend on host/network firewalling and the absence of unsafe WAN port forwarding. Exporters have been tightened separately because they have no reason to be reachable through every host interface.

## Exporter Hardening Changes

### PVE Exporter

The PVE exporter previously published:

```yaml
ports:
  - "127.0.0.1:9221:9221"
```

Prometheus already reaches the exporter as `pve-exporter:9221` over the monitoring Compose network, so a host-published port is unnecessary. The host port has therefore been removed.

Result:

```text
Prometheus container -> pve-exporter:9221 -> Proxmox API
```

There is no longer a host listener for port `9221`.

### node-exporter

node-exporter uses host networking because it is collecting host metrics. Instead of listening on every interface:

```text
:9100
```

it now listens on the Docker host's LAN address:

```text
192.168.2.131:9100
```

This avoids unintentionally creating a listener on other host interfaces such as Tailscale.

### cAdvisor

cAdvisor still needs a host-published endpoint because the Prometheus stack currently scrapes the Docker host by its LAN address. Its port mapping is therefore restricted from:

```text
8080:8080
```

to:

```text
192.168.2.131:8080:8080
```

This keeps the endpoint on the intended LAN interface rather than publishing it on every interface.

## Host Firewall Recommendation

Interface binding reduces accidental exposure but is not a replacement for firewall policy.

A stronger final state is to allow exporter ports only from the monitoring system/required trusted subnet. Because Prometheus currently runs on the same Docker host, firewall rules should be tested carefully so Docker bridge/NAT traffic is not accidentally blocked.

Administrative ports (`3000`, `3001`, `9090`) can also be restricted further once the desired LAN/Tailscale access pattern is finalized.

## Prometheus and Exporter Data

Metrics endpoints can reveal operational information such as:

- hostnames;
- kernel/system information;
- filesystem/device names;
- container names and resource usage;
- virtual machine/container inventory;
- service uptime and capacity.

That information is useful internally but should be treated as infrastructure metadata rather than public website content.

## Proxmox API Credentials

Use a dedicated read-only account/API token for the PVE exporter. Avoid using `root@pam` or an administrative token for metrics collection.

The token value is stored only in:

```text
stacks/monitoring/secrets/pve.yml
```

which is ignored by Git.

## Internet-Facing Services

The planned reverse-proxy/Cloudflare Tunnel phase should distinguish between:

- **public application endpoints** that intentionally need Internet access;
- **administrative endpoints** that should remain LAN/Tailscale-only;
- **metrics/exporter endpoints** that should remain internal.

Prometheus, cAdvisor, node-exporter, and PVE exporter should not be exposed as public Internet services.

## Git Repository Hygiene

The current repository ignores:

- `.env` files;
- `secrets/` directories;
- `*.key` and `*.pem` files;
- runtime application data;
- logs;
- backups.

Before changing a private GitHub repository to public, scan the full history as well as the current tree. Removing a secret from the newest commit does not remove it from previous commits.
