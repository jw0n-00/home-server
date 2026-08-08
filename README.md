# Home Server / Self-Hosted Homelab

Repurposed an old laptop into an always-on Debian server running a small stack of self-hosted services behind a hardened network setup: Docker, Portainer, Jellyfin, Pi-hole, Homepage, Tailscale, UFW, and Fail2Ban.


![Server hardware](screenshots/01-asus-x550la-hardware.jpg)

## Hardware

| Component | Spec |
|---|---|
| Model | ASUS X550LA |
| CPU | Intel Core i5-4200U @ 1.6GHz (2.3GHz Turbo) |
| RAM | 4GB |
| Storage | 512GB |
| OS | Debian 13 (Trixie), headless |

Nothing special on paper, but a spare dual-core laptop is more than enough to run DNS filtering, a media server, and a couple of lightweight dashboards. The goal wasn't raw power, it was proving a full self-hosted stack on hardware that would otherwise be sitting in a drawer.

<p>
  <img src="screenshots/01-asus-x550la-hardware.jpg" width="45%" alt="Laptop bottom panel"/>
  <img src="screenshots/02-asus-x550la-hardware.jpg" width="45%" alt="Laptop lid"/>
</p>

## Why This Project Matters

This isn't just a media server for movies. Every piece of it maps directly to real IT/helpdesk and SOC-analyst work:

- **Linux system administration** — installing, configuring, and maintaining a headless Debian box over SSH
- **Container orchestration** — deploying and managing multiple services with Docker Compose and Portainer instead of installing everything bare-metal
- **DNS and network troubleshooting** — diagnosing why Pi-hole was silently dropping requests by reading logs, not guessing
- **Network security fundamentals** — using a firewall (UFW), intrusion prevention (Fail2Ban), and a zero-exposed-ports VPN (Tailscale) instead of forwarding ports on the router
- **Config management under pressure** — a single mistyped YAML key took down an entire service; catching and fixing that fast is the same skill as triaging a broken deployment at work

In a home environment this stack gives ad-blocking for every device on the network, a private media server, and secure remote access with zero public attack surface. In a professional context, it's a sandbox for practicing the exact troubleshooting and hardening workflows a help desk, NOC, or SOC role expects on day one.

## OS Install — Debian 13

Installed Debian 13 as a minimal server: no desktop environment, SSH enabled from the start so it could be administered headlessly right after setup.

**Boot menu**

![Debian installer boot menu](screenshots/04-debian-boot-menu.jpg)

**Guided partitioning across the full disk**

![Guided partitioning, entire disk](screenshots/07-guided-use-entire-disk.jpg)

**Single-partition layout** — kept simple, no need for separate `/home`, `/var`, or `/tmp` on a single-purpose server

![Partitioning scheme selection](screenshots/08-partition-layout.jpg)

**Software selection** — desktop environments deselected, only SSH server and standard system utilities installed to keep the base OS lightweight

![Software selection screen](screenshots/09-software-selection.jpg)

**First boot to a login prompt**

![Debian login prompt](screenshots/10-debian-login.jpg)

## Finding the Server and Connecting Remotely

With no monitor attached day-to-day, the server needed a known address on the LAN so it could be reached over SSH from another machine.

```
ip addr
```

![ip addr showing the server's LAN address](screenshots/11-ip-addr-to-find-ip-for-server.jpg)

From there, connected in over SSH from another PC on the same network:

![First SSH login to the server](screenshots/10-first-login.jpg)

## Docker Setup

Installed Docker Engine and the Docker Compose plugin, then added my user to the `docker` group so containers can be managed without prefixing every command with `sudo`.

![Docker installed and verified](screenshots/16-docker-working.png)

![Docker Compose plugin installed and verified](screenshots/17-docker-compose.png.png)

All containers live under a single `~/docker` directory, one subfolder per service, each holding its own `docker-compose.yml` and persistent volumes. Keeping every service isolated in its own folder makes it easy to bring one service up or down without touching the others.

![~/docker folder created as the base for every service](screenshots/18-docker-folder-structure.png)

## Deployed Services

### Jellyfin — media server

Runs as a container with dedicated volumes for movies, shows, and music, managed through Portainer for a web-based view of container health.

![Deploying the Jellyfin container](screenshots/21-jellyfin-container.png)

![Jellyfin admin dashboard](screenshots/22-jellyfin-dashboard.png)

### Pi-hole — network-wide ad blocking

Acts as the DNS resolver for the whole network, blocking ad and tracker domains before they ever reach a device.

![Deploying the Pi-hole container](screenshots/26-pihole-container.png)

![Pi-hole dashboard showing blocked queries](screenshots/27-pihole-dashboard.png)

### Homepage — unified dashboard

A single landing page linking out to every service running on the box, with live CPU/RAM/disk stats.

![Homepage dashboard](screenshots/23-homepage-dashboard.png)

### Portainer — container management

Web UI for managing every container above without touching the CLI for routine checks.

## Remote Access — Tailscale

Instead of forwarding ports on the home router, which increases the attack surface, the server joins a private [Tailscale](https://tailscale.com/) mesh network. Tailscale is built on WireGuard and creates an encrypted point-to-point connection between devices, so Jellyfin, Portainer, Homepage, Pi-hole, and SSH are all reachable from anywhere without exposing anything to the public internet.

```
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale status
```

![Tailscale connected and assigned an address](screenshots/24-tailscale-status.png)

> **Note on `curl | sh`:** piping a downloaded script straight into a shell means trusting the source completely, whatever the script contains runs with your permissions. For a known vendor like Tailscale this is a common install pattern, but the safer version for anything less trusted is to download first (`curl -fsSL <url> -o install.sh`), read the script, then run it (`sudo sh install.sh`).

## Hardening the Host

### UFW (firewall)

Enabled UFW and explicitly allowed only SSH (22/tcp), denying everything else by default.

![UFW enabled, SSH allowed, everything else denied](screenshots/15-firewall-status.png)

### Fail2Ban (brute-force protection)

Watches auth logs and temporarily bans IPs that fail login attempts repeatedly, running as an enabled system service.

![Fail2Ban enabled and running](screenshots/14-fail2ban-running.png)

## Final Verification

All four services confirmed healthy and running together:

```
$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED        STATUS                 PORTS                                                                                                                                         NAMES
e3dc57a80f21   pihole/pihole:latest                  "start.sh"               3 hours ago    Up 2 hours (healthy)   67/udp, 0.0.0.0:53->53/tcp, 0.0.0.0:53->53/udp, [::]:53->53/tcp, [::]:53->53/udp, 123/udp, 443/tcp, 0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   pihole
3bb9a434e866   ghcr.io/gethomepage/homepage:latest   "docker-entrypoint.s…"   4 hours ago    Up 2 hours (healthy)   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp                                                                                                   homepage
96c3ff637738   jellyfin/jellyfin                     "/jellyfin/jellyfin"     24 hours ago   Up 2 hours (healthy)   0.0.0.0:8096->8096/tcp, [::]:8096->8096/tcp                                                                                                   jellyfin
521c952e4759   portainer/portainer-ce                "/portainer"             25 hours ago   Up 2 hours             8000/tcp, 9000/tcp, 0.0.0.0:9443->9443/tcp, [::]:9443->9443/tcp                                                                               portainer
```

![All four containers healthy in Portainer](screenshots/28-portainer-containers.png)

Media Automation Pipeline

Extended the stack with a set of containers that automate media library management end-to-end: a request goes in through a web front-end, gets picked up by a library manager, routed through an indexer aggregator, handled by an isolated download client, and lands in Jellyfin's library automatically — no manual file handling at any point.

Services added:

Service	Role
Gluetun	VPN gateway container; isolates the download client's network traffic behind a WireGuard tunnel with an automatic kill switch
qBittorrent	Download client; runs with no network of its own — all traffic is forced through Gluetun
Prowlarr	Centralized indexer/source manager; syncs available sources to the library managers below instead of configuring each one separately
Radarr / Sonarr / Lidarr	Library managers for movies, TV/anime, and music respectively — monitor for requests, coordinate acquisition, and rename/move finished files into the correct Jellyfin library folder
Bazarr	Subtitle automation; watches the Radarr/Sonarr libraries and fetches matching subtitles
FlareSolverr	Headless-browser proxy that resolves anti-bot/JS-challenge pages so Prowlarr can reach sources that require it
Jellyseerr	Request front-end; lets a user search for a title and submit a request that flows automatically to Radarr/Sonarr without touching the backend tools directly

Architecture decisions worth calling out:

Network-scoped VPN isolation, not a blanket VPN. Rather than tunneling the whole server's traffic, only the download client's container network is routed through the VPN gateway (network_mode: "service:gluetun" in Compose) — every other service keeps its normal network path. This required understanding Docker's shared-network-namespace model: qBittorrent has no IP of its own on the Docker network, so every other container that needs to reach it addresses Gluetun's container name instead.
Kill-switch by design. Gluetun tears down network access entirely if the VPN tunnel drops, rather than allowing the download client to silently fall back to a direct connection. Verified this by comparing the external IP reported from inside the Gluetun container against the download client's container — confirming they match, and that neither matches the server's real IP.
Single indexer-management layer. Prowlarr centralizes source configuration and pushes it to every library manager via its Applications sync feature, instead of duplicating that configuration three times across Radarr/Sonarr/Lidarr.
Path consistency across containers. All library managers and the download client mount the same host directories at matching container paths — the single most common failure point in a stack like this, since a library manager can't import a finished file it can't see under the path it expects.

## Troubleshooting Log

Real problems hit during setup, and how each one was diagnosed and fixed instead of just reinstalling.

### 1. Pi-hole silently ignoring every device on the network

**Symptom:** devices on the LAN weren't picking up Pi-hole as their DNS server, queries were dropping.

**Diagnosis:** checked the container logs and found the actual reason:
```
dnsmasq: ignoring query from non-local network <client-ip>
```
Pi-hole's DNS interface was set to **listen on local interface only**, meaning it would answer queries from itself but treated every other device on the LAN as untrusted and refused to respond.

**Fix:** in **Settings → DNS → Interface listening behavior**, switched to **Permit all origins**, then restarted the container:
```
docker restart pihole
```
Verified the fix from a client device:
```
nslookup google.com <server-ip>
```
which returned a response from Pi-hole instead of timing out, and the Query Log immediately started showing traffic from every device on the network.

### 2. Homepage rejecting all browser requests

**Symptom:** loading the Homepage dashboard in a browser returned a host-validation error instead of the dashboard.

**Diagnosis:** Homepage has a built-in host-validation security feature, it only accepts requests where the `Host` header matches an explicitly allowed list. Without that list set, it rejects every request, including legitimate ones from the LAN.

**Fix:** added the server's address to the compose file's environment block:
```yaml
environment:
  HOMEPAGE_ALLOWED_HOSTS: <server-ip>:3000
```
Recreated the container to apply the change:
```
docker compose down
docker compose up -d
```

### 3. YAML typo blocking the container from starting at all

While making the fix above, Docker Compose refused to start the stack:
```
validating docker-compose.yml: services.homepage additional properties 'enviroment' not allowed
```
A one-letter typo (`enviroment` instead of `environment`) was enough for Compose to reject the entire file before the container even attempted to start. Corrected the key name and re-ran `docker compose config` to validate the file before bringing the stack back up.

## Mistakes & Known Limitations

Being upfront about what went wrong and what's still missing:

- **Typo'd a YAML key and lost time to it.** `enviroment` instead of `environment` in the Homepage compose file stopped the whole service from starting. Now I run `docker compose config` before every `up` to catch this kind of thing before it wastes time.
- **Didn't record the server's static IP up front.** Had to run `ip addr` after the fact to rediscover it. A DHCP reservation or static IP set at install time would have saved a step.
- **Pi-hole's default DNS listening mode wasn't LAN-ready out of the box.** I assumed it would "just work" for every device on the network and had to debug logs to find out why it didn't.
- **Everything is still password-auth over SSH, not key-based.** Fine on a Tailscale-only network with no forwarded ports, but SSH key authentication is the next hardening step.
- **No reverse proxy or TLS.** All services are plain HTTP, protected only by being on a private Tailscale network rather than the public internet. Adding something like Caddy or Nginx Proxy Manager with real certs is on the list.
- **No automated backups.** Docker volumes and configs live only on this one disk right now. A dead drive means rebuilding from scratch. Scheduled backups to a second location are the next priority.
- **Single point of failure.** One laptop, one power source, one disk. Acceptable for a homelab, not acceptable for anything that actually needs to stay up.

## What I Learned

- A service that looks broken is often a security feature doing exactly what it's configured to do. Pi-hole rejecting LAN traffic and Homepage rejecting unknown hosts were both intentional defaults, not bugs, and the fix is finding the right setting, not fighting the software.
- Config validation happens before anything runs. One misspelled key in a Compose file stopped an entire service, before Docker even tried to pull an image.
- Not exposing ports beats securing exposed ports. Tailscale means there's no public attack surface for any of these services at all, instead of trying to harden five different open ports individually.
- Defense in depth still matters behind a VPN. UFW and Fail2Ban got configured anyway, because a single layer of protection is still a single point of failure.
- Documentation as you go beats documentation after the fact. Screenshotting each step while building this made writing this README a lot faster, and it's the same habit that makes incident write-ups useful at a real job.

## Stack Overview

| Layer | Tool |
|---|---|
| OS | Debian 13 |
| Containerization | Docker + Docker Compose |
| Container management | Portainer |
| Media server | Jellyfin |
| Ad/tracker blocking (DNS) | Pi-hole |
| Dashboard | Homepage |
| Remote access | Tailscale (WireGuard) |
| Firewall | UFW |
| Intrusion prevention | Fail2Ban |

## Repo Structure

```
homelab/
├── README.md
└── screenshots/
    ├── 01-asus-x550la-hardware.jpg
    ├── 02-asus-x550la-hardware.jpg
    ├── 04-debian-boot-menu.jpg
    ├── 07-guided-use-entire-disk.jpg
    ├── 08-partition-layout.jpg
    ├── 09-software-selection.jpg
    ├── 10-debian-login.jpg
    ├── 10-first-login.jpg
    ├── 11-ip-addr-to-find ip for server.jpg
    ├── 14-fail2ban-running.png
    ├── 15-firewall-status.png
    ├── 16-docker-working.png
    ├── 17-docker-compose.png.png
    ├── 18-docker-folder-structure.png
    ├── 21-jellyfin-container.png
    ├── 22-jellyfin-dashboard.png
    ├── 23-homepage-dashboard.png
    ├── 24-tailscale-status.png
    ├── 26-pihole-container.png
    ├── 27-pihole-dashboard.png
    └── 28-portainer-containers.png
```
