# Troubleshooting Log

A running record of problems encountered during the lab, how they were diagnosed, and how they were resolved. Kept here because working through problems is part of the learning — and because remembering what fixed something last time saves hours next time.

---

## How to use this log

Each entry follows the same structure:

- **Symptom** — what I observed (error message, unexpected behaviour)
- **Service** — which container or system component was involved
- **Diagnosis** — how I figured out what was wrong
- **Fix** — what actually resolved it
- **Lesson** — what I'd check first next time

---

## Entries

---

### [2025-10-13] Port 53 conflict — Pi-hole fails to start

**Service:** Pi-hole
**Session:** 04

**Symptom:**
`docker compose up -d` ran without error, but Pi-hole's admin dashboard was unreachable. `docker logs pihole` showed:

```
Error starting userland proxy: listen tcp4 0.0.0.0:53: bind: address already in use
```

**Diagnosis:**
Ubuntu's `systemd-resolved` service runs a local DNS stub listener that binds to port 53 by default. Pi-hole also needs port 53 for DNS. They were competing for the same port.

```bash
sudo ss -tulpn | grep :53
# Output: LISTEN 127.0.0.53:53  users:(("systemd-resolve",...))
```

**Fix:**

```bash
sudo nano /etc/systemd/resolved.conf
```

Set `DNSStubListener=no` and `DNS=127.0.0.1`, then:

```bash
sudo systemctl restart systemd-resolved
docker compose down && docker compose up -d
```

**Lesson:** Before assuming a port conflict is between two containers, check what the OS itself is using. `ss -tulpn | grep :<port>` is the first diagnostic step for any "address already in use" error.

---

### [2025-10-13] Pi-hole running but query log empty

**Service:** Pi-hole
**Session:** 04

**Symptom:**
Pi-hole admin dashboard accessible. No errors in `docker logs`. But query log showing zero traffic despite browsing the web.

**Diagnosis:**
`/etc/resolv.conf` still pointed at `127.0.0.53` (the old systemd-resolved stub address). DNS queries were not actually going through Pi-hole.

```bash
cat /etc/resolv.conf
# nameserver 127.0.0.53  ← wrong, not pointing at Pi-hole
```

**Fix:**

```bash
sudo nano /etc/resolv.conf
# Changed nameserver to 127.0.0.1
```

Queries appeared immediately in the Pi-hole log.

**Lesson:** Deploying a service and routing traffic through it are two separate steps. Always verify the system is actually using the service you just deployed, not just that the service is running.

---

### [2025-10-20] File Browser — uploaded files disappear after container restart

**Service:** File Browser
**Session:** 05

**Symptom:**
Uploaded a test file through the File Browser web interface. Restarted the container with `docker compose down && docker compose up -d`. File was gone.

**Diagnosis:**
The Docker Compose file was missing a volume definition. Files written inside the container's filesystem are not persisted — they exist only for the lifetime of the container. Without a volume mount, data is lost when the container stops.

**Fix:**
Added a named volume to the compose file:

```yaml
services:
  filebrowser:
    volumes:
      - ./data:/srv          # maps host directory to container storage path
      - ./filebrowser.db:/database/filebrowser.db
```

Recreated the container. Uploaded a test file. Restarted. File persisted.

**Lesson:** Any service that stores data (files, databases, config) needs a volume mount. Check for missing volumes first whenever data disappears after a restart.

---

### [2025-10-27] Vaultwarden — HTTPS warning in Bitwarden app

**Service:** Vaultwarden
**Session:** 07

**Symptom:**
Vaultwarden container running successfully. Admin interface accessible on the local network. When trying to connect the Bitwarden mobile app, connection refused or security warning about non-HTTPS.

**Diagnosis:**
Bitwarden clients (mobile and desktop) require HTTPS connections for security. Vaultwarden running on plain HTTP (`http://192.168.1.105:8081`) is rejected by the client by default.

**Notes at the time:**
HTTPS requires a reverse proxy with a valid certificate. This is not configured until Session 9 (Nginx Proxy Manager). During Session 7, Vaultwarden is accessed only through the browser with the security warning manually acknowledged — **using test credentials only, never real passwords**.

**Fix (applied in Session 9):**
Added Vaultwarden as a proxy host in Nginx Proxy Manager. Issued a Let's Encrypt certificate via Cloudflare DNS challenge. Updated the `DOMAIN` environment variable in Vaultwarden's compose file to the HTTPS URL.

**Lesson:** Some services enforce or strongly expect HTTPS — especially anything handling credentials. Plan for reverse proxy + HTTPS as a dependency, not an afterthought.

---

### [2025-11-03] Uptime Kuma — monitor shows DOWN for running service

**Service:** Uptime Kuma
**Session:** 08

**Symptom:**
Uptime Kuma dashboard showing Pi-hole as DOWN despite Pi-hole running correctly and DNS working.

**Diagnosis:**
The monitor was configured to check `http://pihole:8080` using the container name as the hostname. Uptime Kuma and Pi-hole were in different Docker Compose stacks and not on the same Docker network — so the hostname `pihole` could not be resolved.

```bash
# Checked which networks each container was on
docker inspect pihole | grep NetworkMode
docker inspect uptime-kuma | grep NetworkMode
# Different networks
```

**Fix (option 1 — use IP address):**
Changed the monitor URL to `http://192.168.1.105:8080` (the VM's LAN IP). This works regardless of Docker networks.

**Fix (option 2 — shared network, applied later):**
Created a shared Docker network (`lab-network`) and added both containers to it. Container name resolution then worked correctly.

**Lesson:** Container name resolution only works between containers on the same Docker network. When monitoring services across separate Compose stacks, use the host IP or create a shared external network.

---

### [2025-11-17] Nginx Proxy Manager — 502 Bad Gateway on proxied service

**Service:** Nginx Proxy Manager
**Session:** 09

**Symptom:**
Created a proxy host for File Browser in Nginx Proxy Manager. Navigating to the configured domain returned a `502 Bad Gateway` error.

**Diagnosis:**
502 means Nginx received the request but couldn't reach the upstream service. Common causes:

1. Wrong IP or port in the proxy host config
2. Container not running
3. Nginx container and target container on different networks

Checked in order:

```bash
# 1. Confirmed File Browser was running and on the right port
docker ps | grep filebrowser
curl http://127.0.0.1:8080   # returned HTML — service is up

# 2. Checked Nginx Proxy Manager logs
docker logs nginx-proxy-manager | tail -20
# Showed: connect() failed (111: Connection refused) while connecting to upstream
```

The issue was that Nginx Proxy Manager was trying to reach File Browser by container name, but they were on different networks.

**Fix:**
Added both containers to the shared `lab-network` Docker network. Updated the proxy host in NPM to use the File Browser container name (`filebrowser`) as the forward hostname instead of the VM IP.

**Lesson:** 502 errors from Nginx always mean "I got the request but couldn't reach the backend." Start by confirming the backend is running, then check whether Nginx can actually reach it (network, correct port, correct hostname).

---

## Quick reference — common commands

```bash
# See what's using a port
sudo ss -tulpn | grep :<port>

# Check container logs
docker logs <container-name>
docker logs <container-name> --tail 50    # last 50 lines
docker logs <container-name> -f           # follow live

# List running containers
docker ps

# List all containers including stopped
docker ps -a

# Inspect container network config
docker inspect <container-name> | grep -A 20 Networks

# Restart a single container
docker compose restart <service-name>

# Recreate containers after config change
docker compose down && docker compose up -d

# Test DNS resolution
dig <domain> @<dns-server-ip>
dig ads.doubleclick.net @127.0.0.1     # should return 0.0.0.0 if Pi-hole is blocking

# Test HTTP response
curl -I http://<ip>:<port>
```

---

*Last updated: 2025-11-17 — entries added as problems are encountered*
