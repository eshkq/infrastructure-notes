# homelab-docs

Personal infrastructure documentation. I run a distributed self-hosted environment across 10+ nodes and 40+ containerised services — this repo is where I document what I build, how it works, and why certain decisions were made.

Not polished tutorials. Real notes from real setups, including the mistakes.

---

## About me

Infrastructure-focused engineer with a background spanning QA, systems analysis, and hands-on server administration. I operate a multi-host private infrastructure (Proxmox bare-metal + several VPS nodes) as my primary hobby and learning environment.

Currently working as a Systems Analyst at a software company, acting as the de facto infrastructure owner. Targeting a **DevOps / Infrastructure Engineer** role at a European company.

**Core stack:** Linux · Docker · Proxmox VE · TrueNAS · Traefik · WireGuard/Netbird · Authentik · Grafana · Bash

**Looking for:** Remote or relocation (EU) · English B2 (technical fluency)

---

## Docs

| Document | Description |
|---|---|
| [restic-backup.md](./restic-backup.md) | Encrypted backups to S3-compatible storage with restic — PostgreSQL via stdin pipe + Docker volume directory |

More coming as I build and document things.

---

## Infrastructure overview

```
Proxmox bare-metal (home)
├── VMs and LXC containers
├── TrueNAS — ZFS storage, SMB/NFS shares
└── Docker host — Authentik, Vaultwarden, Grafana,
                  Traefik, CrowdSec, Immich, n8n, ...

VDS (hosted)
└── Docker host — Mailcow, Forgejo, Matrix,
                  VictoriaMetrics, Traefik, ...

Gateway node
└── Netbird coordination server, Authelia, Traefik

All nodes connected via WireGuard overlay (Netbird)
```

---

## Philosophy

- Self-hosted over SaaS where it makes sense
- Privacy-first — data stays on infrastructure I control
- Everything in Git — configs, compose files, docs
- Document as you go — if it took time to figure out, write it down
