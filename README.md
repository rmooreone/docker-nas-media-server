# Homebase Media Server & Automated DevOps Lab

A Docker Compose stack for a home media/download server, running on an Ubuntu Server host. It isolates P2P download tools behind a WireGuard VPN container, serves media with Jellyfin, shares files over Samba, and exposes the server remotely over an encrypted Tailscale mesh.

---

## 🛠️ Architecture Overview

* **Jellyfin** — media server for video and music libraries. Config, cache, and database live on the local system drive (`/var/lib/jellyfin`) rather than the network storage pool, for reliable I/O performance.
* **Samba** — LAN file sharing for Windows/macOS clients, backed by a large storage pool at `/media/pool/samba_shares`.
* **Gluetun** — WireGuard VPN gateway. qBittorrent, Prowlarr, and FlareSolverr all run with `network_mode: container:gluetun`, so their traffic is routed exclusively through the VPN tunnel.
* **Tailscale** — private mesh VPN for secure remote access to the server without exposing ports on your router/firewall.
* **Sonarr / Radarr / Prowlarr / Jellyseerr** — automated media search, request, and download management (the *arr stack).

### Design notes
* **Sensitive service ports are bound to localhost only** (`127.0.0.1`) in the compose file — qBittorrent's WebUI, Prowlarr, and FlareSolverr are not reachable from your LAN by default. Reach them remotely via Tailscale (e.g. `tailscale ssh` + local port-forward, or `tailscale serve`), not by opening them to the network.
* **FlareSolverr has no authentication.** It's only ever called internally by Prowlarr and should never be exposed publicly.
* **Hardlinking**: qBittorrent, Sonarr, and Radarr all mount the same host directory (`/media/pool/samba_shares/public/data`) at the identical container path `/data`. This is what makes hardlinks possible — the *specific* subfolder names don't matter, as long as both the download save path and the library import path live somewhere under that shared `/data` mount. For example, downloading to `/data/media/torrents` and importing into `/data/media/tv` or `/data/media/movies` all works, since they're on the same filesystem. What breaks hardlinking is a download path that resolves to a *different* mount (e.g. the local system drive) than the library path.

---

## 📂 Storage & Family Folder Layout

The Samba server exposes a set of public and private shares from a single host (`\\homebase`):

### Public Spaces (all verified network users)
* `Public_Data` (`/mount/public/data`) — movies, TV, and automated downloads; mapped into Jellyfin's library.
* `Public_Photos` (`/mount/public/photos`) — shared photo library.
* `Public_Music` (`/mount/public/music`) — shared audio library.
* `Public_Documents` (`/mount/public/documents`) — shared household documents.

### Private Spaces
* `Chief_Private` (`/mount/personal/chief`) — restricted to the primary admin login.
* `Family_Vault` (`/mount/family_vault`) — restricted to secondary household logins.

---

## 🚀 Deployment

### 1. Storage provisioning
Run these on the Ubuntu host before starting the stack:

```bash
# Combine data drives with MergerFS
sudo mergerfs -o defaults,allow_other,use_ino,category.create=mfs,minfreespace=20G /mnt/data1:/mnt/data2 /media/pool

# Local SSD cache paths for Jellyfin
sudo mkdir -p /var/lib/jellyfin/config /var/lib/jellyfin/cache

# Storage layout on the pool
sudo mkdir -p /media/pool/samba_shares/{public/data,public/photos,public/music,public/documents,family_vault,personal/chief}

# Ownership and permissions
sudo chown -R 1000:1000 /var/lib/jellyfin/ /media/pool/samba_shares/
sudo chmod -R 775 /var/lib/jellyfin/ /media/pool/samba_shares/
```

### 2. Kernel networking flags
Required for the VPN/overlay networking to route correctly:

```bash
sudo sysctl -w net.ipv4.ip_forward=1 net.ipv6.conf.all.forwarding=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
```

### 3. Configure secrets
```bash
cp .env.example .env
# edit .env with your real VPN keys, timezone, and Samba passwords
```
`.env` is gitignored — never commit it.

### 4. Start the stack
```bash
sudo docker compose up -d
```

### 5. Remote access via Tailscale
Rather than a global Funnel wildcard, scope access deliberately:

```bash
# Expose only Jellyfin's port publicly, if you want it internet-accessible
sudo tailscale funnel --bg http://<tailscale-ip>:8096
```

For the other services (qBittorrent, Prowlarr, Samba shares), prefer keeping them reachable only over your **tailnet** (not Funneled to the public internet), and scope your Tailscale ACLs to specific users or device tags rather than a wildcard `target: ["*"]`. A wildcard target opens the node to every device on your tailnet, not just the one port you intended to publish.

---

## 🚀 Post-Deployment Verification

1. Open the Prowlarr dashboard (via Tailscale, `http://<tailscale-ip>:9696`) and add your preferred indexers.
2. Request a title in Sonarr or Radarr — it queries Prowlarr and sends the result to qBittorrent.
3. qBittorrent downloads into `/data/media/torrents` inside the shared pool mount.
4. Once complete, Sonarr/Radarr import the file into `/data/media/tv` or `/data/media/movies` using a hardlink (near-instant, no duplicated disk usage), since all three containers share the same underlying `/data` mount.

---

## 🔒 Client Access Reference

| User | Access | Windows/macOS Path |
| :--- | :--- | :--- |
| `chief` | All `Public_*` shares + `Chief_Private` | `\\<tailscale-ip>\Chief_Private` |
| `family` | All `Public_*` shares + `Family_Vault` | `\\<tailscale-ip>\Family_Vault` |

---

## ⚠️ Before You Deploy

* Replace all values in `.env` — do not reuse the example placeholders.
* This compose file pins images to `:latest`. That's convenient for a homelab but means an upstream update can change behavior without warning; pin specific version tags if you want reproducible upgrades.
* Review your Tailscale ACLs before enabling Funnel — see the networking note above.
