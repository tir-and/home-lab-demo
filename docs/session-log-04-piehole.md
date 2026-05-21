# Session Log 04 — DNS and Ad Blocking with Pi-hole

**Date:** 2025-10-13
**Programme:** Semester 1 — Infrastructure Lab Core
**Session goal:** Understand how DNS works, deploy Pi-hole as a local DNS server and ad blocker, and read the query logs to see what devices on the network are resolving.

---

## What I built

Deployed Pi-hole as a Docker container on the Ubuntu Server VM. Configured the VM to use Pi-hole as its DNS resolver. Loaded default blocklists and verified that ad domains were being blocked by testing from the host browser.

**Result:** Pi-hole admin dashboard accessible at `http://192.168.1.105:8080/admin`. Query logging showing live DNS traffic. ~12% of queries blocked on first evening.

Screenshot: `screenshots/session-04-pihole-dashboard.png`

---

## Concepts covered

### What DNS actually does

DNS (Domain Name System) is the internet's phone book. When you type `google.com` into a browser, your computer doesn't know the IP address — it asks a DNS server to look it up. The DNS server returns something like `142.250.187.206`, and your browser connects to that IP.

```
Browser → "What's the IP for google.com?"
DNS Server → "142.250.187.206"
Browser → connects to 142.250.187.206
```

Without DNS, you'd need to remember IP addresses for every website. DNS is why domain names work.

### How Pi-hole filtering works

Pi-hole sits between your device and the upstream DNS server (like Cloudflare `1.1.1.1` or Google `8.8.8.8`). When a device asks for a domain, Pi-hole checks it against its blocklist first:

```
Device → "What's the IP for ads.doubleclick.net?"
Pi-hole → checks blocklist → BLOCKED → returns 0.0.0.0
(ad never loads)

Device → "What's the IP for bbc.co.uk?"
Pi-hole → checks blocklist → not blocked → asks upstream DNS → returns real IP
```

This works for any device on the network — phones, laptops, smart TVs — without installing anything on them. You just point their DNS at Pi-hole's IP.

### DNS query logs

The Pi-hole query log shows every DNS request from every device. This is surprisingly useful:

- You can see which domains your devices are reaching out to in the background
- You can spot devices behaving unexpectedly (a smart TV calling home to telemetry servers constantly)
- You start to understand network traffic from a visibility perspective — foundational for security work

---

## Step-by-step: what I ran

```bash
# Created the config directory
mkdir -p ~/configs/pihole && cd ~/configs/pihole

# Created the .env file (not committed to GitHub)
nano .env

# Brought the container up
docker compose up -d

# Checked it was running
docker ps
docker logs pihole
```

Full compose file: `configs/pihole/docker-compose.yml`

---

## Problem I hit — port 53 conflict

**Symptom:** After running `docker compose up -d`, Pi-hole started but DNS resolution on the VM broke completely. `ping google.com` returned `Temporary failure in name resolution`.

**What I tried first:**
- Checked `docker logs pihole` — saw `Error starting userland proxy: listen tcp4 0.0.0.0:53: bind: address already in use`
- This told me port 53 was already in use by something else on the VM

**Diagnosis:**

```bash
# Checked what was using port 53
sudo ss -tulpn | grep :53
```

Output showed `systemd-resolved` was already bound to `127.0.0.53:53`. Ubuntu runs a local DNS stub resolver by default — it occupies port 53, which Pi-hole also needs.

**Fix:**

```bash
# Edited the resolved config
sudo nano /etc/systemd/resolved.conf
```

Changed:
```ini
[Resolve]
# DNSStubListener=yes   ← default (commented out = yes)
```

To:
```ini
[Resolve]
DNSStubListener=no
DNS=127.0.0.1
```

Then:
```bash
sudo systemctl restart systemd-resolved
docker compose down && docker compose up -d
```

Pi-hole came up cleanly. DNS resolution working on the VM. Admin dashboard accessible.

**What I learned:** When a container fails to bind to a port, always check what the OS itself is using that port for — it's not always another container. `ss -tulpn` is now a reflex.

---

## Second problem — queries not showing in the log

**Symptom:** Pi-hole was running and dashboard was accessible, but the query log was empty. No traffic visible.

**Cause:** The VM's DNS was not actually pointing at Pi-hole. The `/etc/resolv.conf` file still referenced the old upstream DNS.

**Fix:**

```bash
sudo nano /etc/resolv.conf
```

Changed:
```
nameserver 127.0.0.53
```

To:
```
nameserver 127.0.0.1
```

Queries immediately started appearing in the log.

**What I learned:** Configuring a service and configuring the system to actually use that service are two separate steps. Easy to deploy something, forget to wire it up, and wonder why nothing is happening.

---

## Verification steps

```bash
# Check Pi-hole container is running
docker ps | grep pihole

# Test DNS resolution through Pi-hole
dig google.com @127.0.0.1

# Test that a blocked domain returns 0.0.0.0
dig ads.doubleclick.net @127.0.0.1
# Expected: 0.0.0.0 (blocked)
```

---

## Portfolio artifacts from this session

- [x] `configs/pihole/docker-compose.yml` — compose file (sanitised)
- [x] `configs/pihole/.env.template` — environment variable template
- [x] `screenshots/session-04-pihole-dashboard.png` — admin dashboard
- [x] `screenshots/session-04-query-log.png` — live query log
- [x] `diagrams/architecture-v2.png` — updated diagram with Pi-hole added
- [x] This session log

---

## Homework completed

- [x] Pi-hole deployed and running
- [x] Blocklists loaded and tested
- [x] Query log showing live traffic
- [x] Architecture diagram updated
- [x] Tested ad blocking in browser (before/after comparison)

---

## What's next

**Session 5 — Private File Services with File Browser**

- Deploy File Browser as a Docker container
- Understand persistent volumes (how data survives container restarts)
- Set up user authentication
- Test upload and download from the host browser
