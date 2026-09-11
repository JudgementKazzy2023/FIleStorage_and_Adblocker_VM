# Personal VM Server — Adblocker + File Storage + Phone Sync

A home-lab project: a virtual machine that acts as a network-wide adblocker (Pi-hole),
a personal file/photo storage server (Nextcloud), and extends both to my phone from
anywhere via a WireGuard-based overlay VPN (Tailscale).

## Architecture

See [`homeserver-architecture.drawio`](./homeserver-architecture.drawio) — open at
[app.diagrams.net](https://app.diagrams.net) or with the VS Code Draw.io extension.

## Stack

| Purpose | Tool |
|---|---|
| Hypervisor | VirtualBox |
| Guest OS | Ubuntu Server 26.04.1 LTS |
| Adblocker | Pi-hole (+ Unbound, planned) |
| File storage | Nextcloud (Docker) |
| Remote access | Tailscale |
| Bulk storage | 1TB external HDD (USB passthrough) |

## Phase 0 — Planning

- Decided on a VM-based build (Type 2 hosted, via VirtualBox) rather than bare-metal Proxmox,
  since I'm running this on my main PC.
- Hardware: VM lives on my `D:` NVMe SSD for performance; a separate 1TB external HDD is
  used purely for bulk file storage, passed through to the VM.

## Phase 1 — Hypervisor & VM Setup

- Installed VirtualBox, created a new VM:
  - 4096 MB RAM, 2 vCPUs
  - 80 GB dynamically allocated virtual disk, stored on `D:` (not `C:`, which lacks space)
  - Networking: Bridged Adapter, so the VM gets its own IP on the home network
- Verified disk location and size in VirtualBox Storage settings before proceeding.

## Phase 2 — Ubuntu Server Install

- Installed **Ubuntu Server 26.04.1 LTS** ("Resolute Raccoon") — used the point-release ISO
  rather than the original 26.04, since it bundles all patches since April 2026 into the
  installer image.
- Install choices:
  - Base: standard "Ubuntu Server" (not minimized — need full logging/tooling for active management)
  - No third-party drivers (not needed on a VM)
  - Storage: guided, entire disk, **LVM disabled** (kept it simple — single ext4 partition +
    EFI partition, no need for LVM's flexible resizing on a single-purpose server)
  - No disk encryption (irrelevant for a local VM file)
  - Skipped Ubuntu Pro for now (standard LTS support is sufficient)
  - **OpenSSH server installed**, password authentication allowed — this is how the server is
    managed going forward, instead of the VirtualBox GUI window
  - Skipped all "Featured Server Snaps" (including the Nextcloud snap — Nextcloud will be
    installed manually via Docker in Phase 4 for more control over data location)
- Post-install:
  - Confirmed static-ish IP via DHCP: `192.168.1.7` (bridged adapter confirmed working)
  - Verified SSH access from host PC: `ssh flashy014@192.168.1.7`
  - Ran `sudo apt update && sudo apt upgrade -y` and `sudo apt autoremove -y` to fully patch
    and clean up the old kernel
  - Noted MAC address (`08:00:27:82:f9:6b`) for setting a router-side DHCP reservation, so
    the IP stays fixed — required for Pi-hole to work reliably as the network's DNS server

## Next — Phase 3

Installing Pi-hole and configuring blocklists targeted at malvertising and app-store
redirect ads.
