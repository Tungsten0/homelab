# Architecture and Topology

This document describes the logical homelab architecture represented by this repository. The repository focuses on compute, storage, container services, and monitoring rather than documenting every physical switch or endpoint on the home network.

## Design Goals

The environment is built around a few practical goals:

- centralize compute on a Proxmox VE host;
- separate file-serving duties from application/container workloads;
- keep important data on redundant ZFS storage;
- use Docker Compose for reproducible self-hosted services;
- provide private remote access through Tailscale;
- collect host, container, and hypervisor metrics in one monitoring stack;
- avoid committing credentials or runtime application state to Git;
- gradually move public-facing workloads behind a controlled reverse-proxy/tunnel layer.

## Logical Network

The primary server-side network is currently on the `192.168.2.0/24` private LAN.

| Host | Role | Address |
| --- | --- | --- |
| `pve.home.arpa` | Proxmox VE hypervisor | `192.168.2.10` |
| `fileserver` | Debian file server container | `192.168.2.127` |
| `docker01` | Debian Docker VM | `192.168.2.131` |

These private addresses are included for topology clarity. They are RFC1918 addresses and are not routable from the public Internet.

```mermaid
flowchart LR
    CLIENTS[Trusted LAN Clients]
    TAIL[Tailscale Clients]
    LAN[192.168.2.0/24]

    CLIENTS --> LAN
    TAIL -. encrypted overlay .-> FILE
    TAIL -. encrypted overlay .-> DOCKER

    LAN --> PVE[pve.home.arpa<br/>192.168.2.10]
    LAN --> FILE[fileserver<br/>192.168.2.127]
    LAN --> DOCKER[docker01<br/>192.168.2.131]
```

## Proxmox Layer

The Proxmox host provides the virtualization layer for the environment.

```mermaid
flowchart TB
    PVE[Proxmox VE]
    CT[CT 110<br/>fileserver]
    VM[VM 100<br/>docker01]

    PVE --> CT
    PVE --> VM
```

The separation is intentional:

- the **file server** owns Samba/storage responsibilities;
- the **Docker VM** owns application containers and monitoring services;
- the **Proxmox host** remains focused on virtualization rather than becoming a general-purpose application server.

## Storage Layer

The file server uses a mirrored ZFS pool for primary shared storage.

```mermaid
flowchart LR
    ZFS[(ZFS Mirror)] --> FS[Debian Fileserver]
    FS --> SMB[Samba Shares]
    SMB --> LANCLIENTS[LAN / Tailscale Clients]
    SMB -. external storage .-> NC[Nextcloud]
```

The mirrored pool provides disk redundancy, while snapshots and backups are maintained separately. Mirroring is therefore not treated as a substitute for backups.

## Docker Host

The Docker VM runs independent Compose stacks rather than one large Compose project. This keeps service lifecycle and troubleshooting boundaries simple.

```mermaid
flowchart TB
    HOST[docker01]

    subgraph NEXTCLOUD[Nextcloud Stack]
        NC[Nextcloud Apache]
        DB[(PostgreSQL)]
        R[(Redis)]
        CRON[Nextcloud Cron]
        NC --> DB
        NC --> R
        CRON --> NC
    end

    subgraph MONITORING[Monitoring Stack]
        P[Prometheus]
        G[Grafana]
        PE[PVE Exporter]
        P --> G
        P --> PE
    end

    subgraph EXPORTERS[Docker Host Exporters]
        NE[node-exporter]
        CA[cAdvisor]
    end

    K[Uptime Kuma]

    HOST --> NEXTCLOUD
    HOST --> MONITORING
    HOST --> EXPORTERS
    HOST --> K

    P --> NE
    P --> CA
```

## Monitoring Data Flow

Prometheus is the collection point for metrics from the Docker host, file server, and Proxmox environment.

```mermaid
flowchart LR
    PROM[Prometheus]
    GRAF[Grafana]
    NODE1[node-exporter<br/>docker01]
    NODE2[node-exporter<br/>fileserver]
    NODE3[node-exporter<br/>Proxmox host]
    CAD[cAdvisor<br/>docker01]
    PVEEXP[PVE Exporter]
    PVE[Proxmox API]

    NODE1 --> PROM
    NODE2 --> PROM
    NODE3 --> PROM
    CAD --> PROM
    PVE --> PVEEXP
    PVEEXP --> PROM
    PROM --> GRAF
```

The PVE exporter is kept inside the monitoring Compose network and is not published as a host-accessible service. Prometheus forwards the target Proxmox node to the exporter using relabeling.

## Service Boundaries

Several services intentionally remain reachable only through internal Docker networks:

- Nextcloud PostgreSQL database;
- Nextcloud Redis service;
- PVE exporter after the current hardening change.

User-facing or administrator-facing services may publish host ports, but the current design assumes they are used from a trusted LAN or a controlled remote-access path.

## Planned Evolution

The next architecture phase is expected to add a controlled ingress layer:

```mermaid
flowchart LR
    INTERNET((Internet)) --> CF[Cloudflare Tunnel]
    CF --> RP[Reverse Proxy]
    RP --> PUBLIC[Approved Public Services]

    LAN[Trusted LAN] --> ADMIN[Administrative Services]
    TAIL[Tailscale] --> ADMIN
```

The goal is to keep administrative interfaces such as Prometheus and infrastructure exporters off direct public ingress while selectively publishing services that actually require Internet access.
