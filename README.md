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

## Static IP Configuration

- Initial DHCP-assigned IP (`192.168.1.7`) turned out to already be in use by another
  device on the network, causing conflicts.
- Switched to configuring a static IP directly on the VM via netplan (rather than a
  router-side DHCP reservation), since it doesn't depend on router UI access:
  - Edited `/etc/netplan/00-installer-config.yaml`, disabled DHCP (`dhcp4: false`),
    set a fixed address, default route via the router (`192.168.1.1`), and public
    DNS resolvers (`8.8.8.8`, `1.1.1.1`) as fallback nameservers.
  - Used `sudo netplan try` before committing, which auto-reverts if the new config
    breaks connectivity — avoided any risk of losing SSH access mid-change.
- **Final static IP: `192.168.1.205`** (moved off `.7` due to the conflict above).
- Verified the change by confirming `ip a` showed the new address and SSH access
  still worked at the new IP.

## Security Hardening

Before installing any services, hardened SSH access and added basic intrusion
protection:

- **SSH key-based authentication**
  - Generated an ED25519 key pair (`ssh-keygen -t ed25519`) on the Windows host,
    protected with a passphrase.
  - Copied the public key to the VM's `~/.ssh/authorized_keys`.
  - Verified key-based login worked *before* disabling passwords, to avoid getting
    locked out.
  - Disabled password authentication in `/etc/ssh/sshd_config`
    (`PasswordAuthentication no`) — note: Ubuntu's cloud-init also drops an
    override file at `/etc/ssh/sshd_config.d/50-cloud-init.conf` that can silently
    re-enable password auth; had to edit that file too since it takes priority
    over the main config.
  - Confirmed the fix by forcing a password-only connection attempt
    (`ssh -o PubkeyAuthentication=no ...`) and verifying it was refused
    ("Permission denied (publickey)").
- **Firewall (ufw)**
  - Enabled with a default-deny posture, explicitly allowing only what's needed:
    OpenSSH (22), DNS (53 tcp/udp), Pi-hole web admin (80), and DHCP (67 udp,
    unused but harmless to allow).
- **fail2ban**
  - Installed with default settings to auto-block IPs after repeated failed SSH
    login attempts.
- **Network exposure policy**
  - No router port-forwarding to this VM. All remote access will go through
    Tailscale (planned for Phase 5) rather than exposing any service directly to
    the internet.

## Phase 3 — Pi-hole Installation & Blocklists

- Installed Pi-hole via the official installer script
  (`git clone --depth 1 https://github.com/pi-hole/pi-hole.git Pi-hole
    cd "Pi-hole/automated install/"
    sudo bash basic-install.sh`), run directly over SSH.
  - Hit a couple of install hangs caused by accidentally interrupting the
    installer mid-run (Ctrl+C) and by adding a debug flag (`bash -x`) that
    corrupted the interactive dialog rendering — resolved by killing the
    orphaned processes (`sudo kill -9 <pid>`) and re-running cleanly.
- Setup choices:
  - Upstream DNS: **Cloudflare (DNSSEC)**
  - Blocklist: default StevenBlack list included during install
  - Query logging: enabled
  - Privacy mode: "Show everything" (full detail for a personal/learning project)
- **Admin dashboard**: `http://192.168.1.205/admin`
- **Added additional blocklists** (HaGeZi's repo was restructured since these
  guides were originally written — verified current URLs before adding):
  - `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/pro.txt` —
    general ads/trackers/malware (~222K domains)
  - `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/tif.mini.txt` —
    threat intelligence feed, sized down from the full 2.1M-entry version to keep
    load reasonable on a 4GB RAM VM (~177K domains)
  - `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/popupads.txt` —
    targeted at pop-up/redirect ads specifically (~49K domains)
  - `https://big.oisd.nl` — broad general-purpose coverage (~245K domains)
- Ran **Tools → Update Gravity** to compile all lists: **774,017 total domains,
  526,698 unique** after dedup.
- **iPhone DNS test**: confirmed manual DNS override
  (Settings → Wi-Fi → (i) → Configure DNS → Manual → `192.168.1.205`) blocks ads
  while on home Wi-Fi. Only covers home network — full protection on cellular/other
  networks is deferred to the Tailscale setup in Phase 5.

## Next — Phase 4

Setting up Nextcloud (Docker) for file/photo storage on the 1TB external HDD.
