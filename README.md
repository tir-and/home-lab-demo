# 🖥️ Home Lab — Self-Hosted Infrastructure

![Status](https://img.shields.io/badge/status-active-brightgreen)
![OS](https://img.shields.io/badge/OS-Ubuntu_24.04_LTS-orange)
![Runtime](https://img.shields.io/badge/runtime-Docker_Compose-blue)
![Programme](https://img.shields.io/badge/programme-Semester_1_In_Progress-yellow)

A personal self-hosted infrastructure lab built as part of the **Practical Self-Hosted Infrastructure Programme**. The goal is to build, document, and progressively secure a real working environment using Linux, Docker, and open-source services — the kind you'd find in a small business or home office.

Every service added to this lab is documented here with architecture context, configuration notes, troubleshooting lessons, and screenshots.

> 💡 **What this lab demonstrates:** Real infrastructure skills — Linux administration, containerisation, DNS, reverse proxying, HTTPS, monitoring, and remote access — all running on a single Ubuntu Server VM.

---

## Architecture Overview

All services run as Docker containers on a single Ubuntu Server VM. The host machine connects via SSH. External access is handled through Cloudflare Tunnel — no ports are exposed directly to the internet.

```
Host Machine (Laptop)
  └── VirtualBox VM — Ubuntu Server 24.04 LTS (4GB RAM)
        └── Docker Compose
              ├── Pi-hole          (DNS + ad blocking)
              ├── File Browser     (private file access)
              ├── Vaultwarden      (password management)
              ├── Immich           (photo backup)
              ├── Uptime Kuma      (monitoring)
              └── Nginx Proxy Mgr  (reverse proxy + HTTPS)
                    └── Cloudflare Tunnel (remote access)
```

Full architecture diagram: `diagrams/architecture-v1.png`

---

## Services Deployed

| Service | Purpose | Status | Port | Notes |
|---|---|---|---|---|
| **Pi-hole** | DNS + ad blocking | ✅ Running | 53 / 80 | ~15k queries/day blocked |
| **File Browser** | Private file access | ✅ Running | 8080 | Behind Nginx reverse proxy |
| **Vaultwarden** | Password management | ✅ Running | 8081 | Local-only, not exposed externally |
| **Immich** | Photo backup | ✅ Running | 2283 | Multi-container (Postgres + Redis) |
| **Uptime Kuma** | Service monitoring | ✅ Running | 3001 | Monitoring all 5 services |
| **Nginx Proxy Manager** | Reverse proxy + HTTPS | ✅ Running | 81 (admin) | Let's Encrypt certs via Cloudflare DNS |
| **Wazuh** | SIEM / log monitoring | 🔜 Semester 2 | — | Planned |
| **CrowdSec** | Intrusion prevention | ⬜ Not started | — | Planned |

---

## Repository Structure

```
home-lab/
├── configs/                  # Docker Compose files and .env templates
│   ├── pihole/
│   ├── immich/
│   └── nginx-proxy-manager/
├── diagrams/                 # Architecture diagrams (PNG)
├── docs/                     # Session logs and troubleshooting notes
│   ├── session-log-01.md
│   ├── session-log-04-pihole.md
│   └── troubleshooting.md
├── screenshots/              # Working service screenshots
└── README.md
```

---

## Session Logs

Each session has a corresponding log in `/docs` covering what was built, what broke, and what was learned.

### Session 4 — Pi-hole (DNS + Ad Blocking)

![Complete](https://img.shields.io/badge/status-complete-brightgreen)

**What I built:** Deployed Pi-hole as a local DNS server. Configured the host VM to use Pi-hole as its DNS resolver. Set up blocklists covering ~1.2 million domains.

**Concepts covered:** DNS query flow, how blocklists work, the difference between upstream DNS and local DNS, reading query logs for visibility.

**Problem I hit:** After starting the Pi-hole container, DNS resolution broke entirely on the VM.

**How I fixed it:** The Ubuntu `systemd-resolved` service was already bound to port 53, conflicting with Pi-hole. Disabled `DNSStubListener` in `/etc/systemd/resolved.conf` and restarted the service. DNS resolved correctly after that.

```ini
# /etc/systemd/resolved.conf — fix for port 53 conflict
[Resolve]
DNSStubListener=no
DNS=127.0.0.1
```

**What I learned:** Ports can be occupied by the OS itself, not just other containers. Checking what's using a port (`ss -tulpn | grep :53`) should always be the first troubleshooting step for port conflict errors.

---

## Key Config Examples

### Pi-hole — docker-compose.yml

```yaml
# configs/pihole/docker-compose.yml
# Note: real credentials are in .env (not committed to this repo)
version: '3'
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80"
    environment:
      WEBPASSWORD: ${PIHOLE_PASSWORD}
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
    restart: unless-stopped
```

> 🔒 **Credential safety:** All `.env` files containing real passwords are listed in `.gitignore` and never committed. Only template files with placeholder values are in this repository.

---

## Environment

| Component | Detail |
|---|---|
| Host OS | Windows 11 / macOS |
| Hypervisor | VirtualBox 7.0 |
| Guest OS | Ubuntu Server 24.04 LTS |
| VM RAM | 4 GB |
| VM Storage | 40 GB (dynamically allocated) |
| Container runtime | Docker 26 + Docker Compose v2 |
| Remote access | Cloudflare Tunnel (free tier) |

---

## What I Can Demonstrate

- SSH into the Ubuntu Server VM and navigate the file system
- Bring services up and down with `docker compose up -d` and `docker compose down`
- Show live DNS query logs in the Pi-hole admin interface
- Show all monitored services with uptime status in Uptime Kuma
- Access a service via its HTTPS domain name through Nginx Proxy Manager
- Explain the architecture and how each service connects to the others
- Walk through a troubleshooting log and explain the diagnostic process

---

*Built as part of the Practical Self-Hosted Infrastructure Programme · Semester 1: Infrastructure Lab Core*
