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

## Phase 4 — Nextcloud (File & Photo Storage)

- Backend database: **PostgreSQL** instead of MariaDB (switched on a friend's
  recommendation — Nextcloud's own docs also recommend Postgres for new installs).
- Passed the 2TB external HDD (initially thought to be 1TB) through to the VM via
  VirtualBox USB passthrough.
  - **Gotcha**: device letters (`/dev/sda` vs `/dev/sdb`) swapped between reboots
    depending on attach order — always re-verified with
    `lsblk -o NAME,SIZE,MODEL,FSTYPE,LABEL` before running any destructive command,
    since blindly trusting the same device letter each time would have risked
    wiping the VM's own OS disk instead of the external drive.
  - Drive already had an existing NTFS filesystem with personal files on it —
    backed everything up to the Windows host via `scp` before reformatting
    (`sudo mkfs.ext4 /dev/sdb1`) to ext4.
  - Mounted at `/mnt/storage`, added a permanent `/etc/fstab` entry keyed by UUID
    (not device letter, since that can change) so it survives reboots.
- Installed **Docker** (`docker.io` package, already bundled modern Compose as a
  plugin — used `docker compose` with a space, not the old standalone
  `docker-compose` hyphenated binary).
- `docker-compose.yml` runs two containers: `db` (postgres:16) and `app`
  (nextcloud), with both data volumes pointed at `/mnt/storage` (`postgres/` and
  `nextcloud/` subfolders) so nothing lands on the VM's 80GB OS disk.
- Opened port `8080/tcp` in ufw for the Nextcloud web UI.
- **Main issue hit**: typo'd mismatched `POSTGRES_PASSWORD` values between the
  `db` and `app` services in the compose file, causing
  `password authentication failed for user "nextcloud"` on setup.
  - Fixing the compose file alone wasn't enough — Postgres only applies
    `POSTGRES_PASSWORD` on a truly empty data directory, so it kept using the
    original (wrong) password from its first run even after the file was
    corrected.
  - Resolved by fully tearing down (`docker compose down -v`) and deleting the
    entire `/mnt/storage/postgres` folder (not just its contents) to force a
    genuine `initdb` on the next `docker compose up -d` — confirmed via the logs
    showing `running bootstrap script` instead of `database system was shut
    down` (which indicated it was reusing old data).
- Skipped Nextcloud's bundled recommended apps (Calendar, Mail, Talk, etc.) on
  install — kept it lean given the VM's 4GB RAM is shared with Pi-hole.
- Verified storage is correctly routed to the external drive, not the OS disk:
  `/mnt/storage` (1.8TB drive) showed actual usage growing; `/` (80GB OS disk)
  stayed flat.
- Changed the admin account password after it was briefly visible on-screen
  during setup.
- Confirmed file upload/download works through the web UI.

## Next — Phase 5

Setting up Tailscale so Pi-hole's adblocking and Nextcloud's file sync both work
on the iPhone from anywhere, not just on home Wi-Fi.

## Phase 5 — Tailscale (Remote Access for iPhone)

- Installed Tailscale on the VM (`curl -fsSL https://tailscale.com/install.sh | sh`
  then `sudo tailscale up`), authorized via the login link in a browser.
- VM's Tailscale IP: `100.115.94.1`.
- Installed the Tailscale app on iPhone, logged into the same account. Confirmed
  the tunnel works both directions with `tailscale ping <phone-name>` from the VM.
- **DNS override (Pi-hole everywhere, not just home Wi-Fi)**:
  - In the [Tailscale admin console → DNS](https://login.tailscale.com/admin/dns),
    added the VM's Tailscale IP (`100.115.94.1`) as a **Global nameserver** and
    enabled **"Override local DNS"**.
  - This replaced the earlier approach of manually setting DNS per Wi-Fi network
    on the phone (Settings → Wi-Fi → Configure DNS), which only worked on that
    one network and proved unreliable/easy to accidentally revert.
  - **Issue hit**: even with the override configured correctly, websites failed
    to load entirely on cellular. Root cause: Pi-hole's default **Interface
    settings** ("Allow only local requests") only accepts DNS queries from
    devices one hop away on the home LAN — Tailscale's tunnel interface doesn't
    count as "local," so it was silently dropping all queries arriving over the
    tunnel.
    - Fixed in Pi-hole dashboard: **Settings → DNS → Interface settings →
      "Permit all origins."** This is normally flagged as a "potentially
      dangerous" option, but is safe here specifically because there's no port
      forwarding on the router and `ufw` is already restricting what can reach
      the VM — the only path in is already-authorized Tailscale devices.
  - Confirmed fix by toggling Tailscale off/on on the iPhone and successfully
    loading sites on cellular data (no home Wi-Fi).
- **Nextcloud over Tailscale**:
  - Nextcloud rejects requests to any address not on its `trusted_domains` list,
    which only had the LAN IP (`192.168.1.205`) by default.
  - Added the Tailscale IP via the Nextcloud container's `occ` CLI:
    ```
    sudo docker exec -it nextcloud-app-1 bash
    php occ config:system:set trusted_domains 1 --value="100.115.94.1"
    ```
  - Re-logged into the iPhone Nextcloud app pointing at
    `http://100.115.94.1:8080` instead of the LAN address.
  - Confirmed Files load and sync works over cellular, not just home Wi-Fi.
- **Result**: both core project goals — network-wide adblocking and phone
  file/photo sync — now work from anywhere, not just at home, fulfilling the
  original project requirements.

## Project Status

Core build complete: Pi-hole (adblocking), Nextcloud (file/photo storage), and
Tailscale (remote access for both) are all working together on the iPhone,
both on home Wi-Fi and cellular. Possible future work: Unbound for fully
self-hosted DNS resolution, automated backups of the Nextcloud/Postgres data,
and extending Tailscale/Pi-hole coverage to other devices (Smart TV, laptop).
