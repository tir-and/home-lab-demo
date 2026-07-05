# Week 1 — Linux Server, SSH & Basic Administration

## What I built

This week I set up the actual server that everything else in this lab is going to run on. It is just a fresh Ubuntu Server virtual machine sitting inside VirtualBox on my own machine — nothing fancy, but it's the foundation for the rest of the semester. From here on, every service I deploy lives inside this VM.

## VM specs

- Name: `ubuntu-lab`
- OS: Ubuntu Server (no desktop, terminal only)
- RAM: 4096 MB
- Disk: 25 GB (to be expanded if needed)
- Network: Bridged Adapter (so the VM gets its own IP on my home network)
- Username: `labuser`

## Getting the IP

Once the VM was up, I ran:

```
ip a
```

and looked for the `inet` line on the network adapter. That gave me the VM's actual address on my network — something like `192.168.1.x`. I wrote it down because I'll need it every time I SSH in.

## Connecting via SSH

Instead of working inside the VirtualBox console window, I switched to SSH from my normal terminal:

```
ssh labuser@192.168.1.x
```

This is honestly the bigger shift this week — moving to "working from a real terminal like a sysadmin would." 

## First commands I ran

Just the basics every sysadmin apparently reaches for constantly:

```
sudo apt update && sudo apt upgrade -y   # update packages
whoami                                    # who am I
hostname                                  # what's this machine called
df -h                                     # disk space
free -h                                   # memory usage
top                                       # running processes
sudo journalctl -n 20                     # last 20 log lines
```

Disk, memory, processes, logs — that's the diagnostic sequence if something behaves oddly.

## One problem I hit and how I fixed it

## Screenshots

- VirtualBox showing the VM running
- SSH terminal session (`labuser@ubuntu-lab`)
- Output of `ip a`
- Output of `df -h` / `free -h`

## Next up

Session 3 — Docker and Docker Compose. Homework is just to come with the VM running.
