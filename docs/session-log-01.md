# Session Log 01 — Orientation and Portfolio Setup

**Date:** 2025-09-15
**Programme:** Semester 1 — Infrastructure Lab Core
**Session goal:** Understand what self-hosting is, set up the GitHub portfolio repository, and create an initial architecture diagram.

---

## What I did this session

- Attended the programme introduction and walked through the full 10-week roadmap
- Created this GitHub repository and set up the folder structure
- Wrote a README with a project description and planned services list
- Started the initial architecture diagram in draw.io

---

## What I learned

**Self-hosting** means running applications on a machine you control rather than relying on third-party cloud services. Instead of using Google Drive, Dropbox, or Bitwarden's hosted service, you run the same type of software on your own server.

This matters for IT and cybersecurity roles because:

- You develop real hands-on experience with Linux, Docker, DNS, and networking
- You build something visible and demonstrable for job applications
- You understand infrastructure from the inside, not just in theory

**Why Docker?** Rather than installing software directly on the server (which can conflict and is hard to undo), Docker runs each service in an isolated container. Think of it like a sealed box — everything the service needs is inside it, and removing the service is as simple as stopping the container.

**Why document everything?** A working lab with no documentation is invisible. A documented lab with screenshots, config files, and troubleshooting notes tells the story of real work. That story is what gets noticed.

---

## Architecture diagram — current state (Week 1)

```
Host Machine (Laptop / Desktop)
  └── VirtualBox
        └── Ubuntu Server VM  ← not built yet, coming in Session 2
```

The diagram will grow each week. By Session 10 it will show a full stack of services, a reverse proxy, HTTPS, and remote access via Cloudflare Tunnel.

Full diagram file: `diagrams/architecture-v1.png`

---

## Homework completed

- [x] GitHub repository created and made public
- [x] Folder structure created (`/configs`, `/diagrams`, `/docs`, `/screenshots`)
- [x] README written with project description
- [x] Initial architecture diagram started in draw.io
- [ ] Read the Docker overview on docs.docker.com *(carry forward to next week)*

---

## Questions I still have

- How much RAM does the Ubuntu Server VM actually need? (answer expected in Session 2)
- Do I need a domain name for the reverse proxy, or can I use a local hostname?
- What happens to the data inside a container if I stop it?

---

## What's next

**Session 2 — Linux Server, SSH, and Basic Administration**

- Install Ubuntu Server in a VirtualBox VM
- Connect to it via SSH from the host machine
- Learn basic Linux navigation and package management
