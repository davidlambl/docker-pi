# docker-pi

Docker Compose stack for a Raspberry Pi 5 running **AdGuard Home** (DNS) and
**UniFi Network Application** (wireless controller), migrated from a Pi 4 and an
N150 mini-PC respectively.

## Architecture

```
LAN clients ──DNS:53──▶ Firewalla ──forward──▶ Pi 5 (AdGuard Home) ──DoH/DoT──▶ NextDNS
UniFi APs ──inform:8080──▶ Pi 5 (UniFi Network Application) ──▶ MongoDB (internal)
```

- **AdGuard Home** runs with `network_mode: host` so it owns port 53 directly.
- **UniFi Network Application** and **MongoDB** share a Docker bridge network
  (`unifi-net`). UniFi ports are mapped to the host explicitly.
- MongoDB is not exposed outside Docker — only reachable by the UniFi container.

See [docs/decisions.md](docs/decisions.md) for full rationale.

---

## Prerequisites

- Raspberry Pi 5 with NVMe drive
- Raspberry Pi OS Lite (64-bit) imaged onto the NVMe
- Ethernet connection to LAN
- Static IP reserved in Firewalla DHCP (ideally reusing the N150's current IP —
  see Phase 2)
- SSH access to the Pi 5 from your workstation

---

## Phase 1 — Capture state from existing boxes

### 1.1 Back up AdGuard Home from the Pi 4

Try in order:

**Plan A — SSH from the N150 (or another LAN host that can reach the Pi 4):**

```bash
# From the N150, SSH into the Pi 4
ssh <pi4-user>@192.168.231.145

# Native install — grab config + data
sudo tar czf /tmp/adguard-backup.tar.gz /opt/AdGuardHome/

# Or if AdGuard was running in Docker, find the volume:
docker inspect adguardhome | grep -A5 Mounts
# Then tar the bound directories instead.

# Copy the archive to the N150
exit
scp <pi4-user>@192.168.231.145:/tmp/adguard-backup.tar.gz ~/
```

Then transfer from the N150 to your Mac (or directly to the Pi 5 later).

**Plan B — SD card extraction (if the Pi 4 is unreachable from all hosts):**

1. Power off the Pi 4 and remove the SD card.
2. Insert into your Mac via a USB reader.
3. macOS cannot read ext4 natively. Options:
   - Start a quick Linux VM (UTM, Multipass, or Docker) and pass through the USB
     device to mount the ext4 partition.
   - Use a Linux machine if one is available.
4. Copy `/opt/AdGuardHome/AdGuardHome.yaml` and `/opt/AdGuardHome/data/` from the
   mounted filesystem.

**What you need from the backup:**

| File / directory                     | Purpose                                  |
|--------------------------------------|------------------------------------------|
| `AdGuardHome.yaml`                   | All settings: upstreams, filters, rewrites, DHCP, clients |
| `data/filters/`                      | Downloaded blocklist snapshots            |
| `data/stats.db`                      | Query statistics                         |
| `data/querylog.json` (or `.jsonl`)   | Query log history                        |
| `data/sessions.db`                   | Login sessions (can be discarded)        |

### 1.2 Back up UniFi from the N150

1. Open the UniFi OS Server web UI on the N150 (typically `https://<n150-ip>`).
2. Navigate to **Settings → System → Backups**.
3. Click **Download Backup** and save the `.unf` file.
4. Note the **UniFi Network Application version** (Settings → System → About).
   The destination Docker image must be **equal to or newer** than this version for
   restore to succeed.
5. Record the current controller IP/hostname and the number of adopted devices for
   a post-migration sanity check.

Place the `.unf` file in `unifi/backups/` in this repo (it is gitignored).

### 1.3 Troubleshooting Mac → Pi 4 connectivity

Your Mac gets "No route to host" while the N150 can ping the Pi 4. This is almost
certainly a VLAN or client-isolation change made in the UniFi network config recently.
Check:

- Are the Mac and Pi 4 on the same VLAN / subnet? (`ifconfig` on Mac, compare subnets)
- Is client isolation enabled on the SSID the Mac is using?
- Is there a firewall rule on the Firewalla blocking inter-device traffic on that
  subnet?

Fixing this is independent of the migration but worth resolving so you can manage the
Pi 5 from your Mac afterward.

---

## Phase 2 — Pi 5 fresh install

### 2.1 Reimage the NVMe

Pick one approach:

- **USB enclosure method:** Remove the NVMe from the Pi 5, put it in a USB enclosure,
  connect to your Mac, and use **Raspberry Pi Imager** to write
  **Raspberry Pi OS Lite (64-bit)**. In Imager's advanced settings (gear icon):
  - Set hostname (e.g. `pi5`)
  - Enable SSH (password or key-based)
  - Set username and password
  - Optionally configure Wi-Fi (for initial headless setup before Ethernet is
    connected)
  - Set locale and timezone
  Then reinstall the NVMe in the Pi 5 and boot.

- **SD card bootstrap method:** Image a microSD with Raspberry Pi OS, boot from it,
  then use `rpi-imager` or `dd` to reimage the NVMe in place. Shut down, remove SD,
  and reboot from NVMe.

### 2.2 Verify NVMe boot order

If the Pi 5 boots successfully from the NVMe in step 2.1, the boot order is already
correct and you can skip this section. You only need this if the Pi 5 **won't boot
from NVMe** and you had to fall back to an SD card to get in.

SSH into the Pi 5 and run:

```bash
sudo rpi-eeprom-config
```

Look for `BOOT_ORDER` in the output. It should contain the value `6` (NVMe). The
default Pi 5 EEPROM already prefers NVMe, but if it's missing, update it:

```bash
sudo rpi-eeprom-config --edit
# Change (or add) the BOOT_ORDER line to:
#   BOOT_ORDER=0xf416
# Save and exit, then reboot:
sudo reboot
```

### 2.3 Static IP

Reserve the Pi 5's MAC address in Firewalla DHCP with a static IP.

**Strong recommendation:** Reuse the N150's current IP address. This lets UniFi APs
re-adopt automatically (their stored inform URL already points to this IP). Power off
the N150 before booting the Pi 5 with the same IP.

---

## Phase 3 — Host bootstrap

Run these commands on the Pi 5 over SSH. Each section is a discrete concern, designed
to map cleanly to an Ansible role later (see
[docs/ansible-future.md](docs/ansible-future.md)).

### 3.1 System update and timezone

```bash
sudo apt update && sudo apt full-upgrade -y
sudo timedatectl set-timezone America/New_York    # adjust to your timezone
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades   # enable auto security updates
```

### 3.2 Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# Log out and back in for the group change to take effect
exit
```

Verify after re-login:

```bash
docker --version
docker compose version
```

### 3.3 Free port 53 for AdGuard Home

AdGuard Home needs to bind port 53. On some distros, `systemd-resolved` runs a stub
listener on that port. Raspberry Pi OS Lite does **not** ship with `systemd-resolved`,
so if you're on Pi OS Lite you can skip straight to setting `/etc/resolv.conf` below.

**Only if `systemd-resolved` is present** (e.g. Ubuntu Server or desktop Pi OS):

```bash
sudo sed -i 's/#DNSStubListener=yes/DNSStubListener=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

**All installs — set a temporary upstream nameserver:**

```bash
sudo rm -f /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

Once AdGuard Home is running, you can optionally point this at `127.0.0.1` so the
Pi itself resolves through AdGuard.

### 3.4 Firewall (optional but recommended)

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH
sudo ufw allow 22/tcp

# AdGuard Home
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 8081/tcp        # AdGuard admin UI (after first-run setup)
sudo ufw allow 3000/tcp        # AdGuard first-run wizard (can remove after setup)

# UniFi Network Application
sudo ufw allow 8443/tcp        # Web admin (HTTPS)
sudo ufw allow 8080/tcp        # Device inform
sudo ufw allow 3478/udp        # STUN
sudo ufw allow 10001/udp       # AP discovery
sudo ufw allow 1900/udp        # L2 discovery (optional)
sudo ufw allow 8843/tcp        # HTTPS portal redirect (optional)
sudo ufw allow 8880/tcp        # HTTP portal redirect (optional)
sudo ufw allow 6789/tcp        # Speed test (optional)

sudo ufw enable
sudo ufw status
```

### 3.5 Clone this repo

```bash
cd ~
git clone https://github.com/<your-username>/docker-pi.git
cd docker-pi
```

Or if you prefer `/opt`:

```bash
sudo mkdir -p /opt/docker-pi
sudo chown $USER:$USER /opt/docker-pi
git clone https://github.com/<your-username>/docker-pi.git /opt/docker-pi
cd /opt/docker-pi
```

---

## Phase 4 — Configure and deploy

### 4.1 Create the `.env` file

```bash
cp .env.example .env
```

Generate two random passwords (copy each output — you'll need them in a moment):

```bash
openssl rand -base64 24
openssl rand -base64 24
```

Open `.env` in an editor:

```bash
nano .env
```

Find these two lines and replace the placeholder values with the passwords you just
generated:

```
MONGO_INITDB_ROOT_PASSWORD=CHANGEME_root_password    ← paste first password
MONGO_PASS=CHANGEME_unifi_password                   ← paste second password
```

The other defaults should be fine as-is:

- `PUID`/`PGID`: should match your user. Run `id` — if you see `uid=1000` and
  `gid=1000`, no change needed.
- `TZ`: set to `America/New_York`. Change if you're in a different timezone.

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

### 4.2 Restore AdGuard Home config (optional)

If you have a backup from Phase 1.1, extract `AdGuardHome.yaml` into the conf
directory:

```bash
# From the backup tarball:
tar xzf adguard-backup.tar.gz -C /tmp
cp /tmp/opt/AdGuardHome/AdGuardHome.yaml ./adguardhome/conf/

# Edit to change the admin UI port (avoid port 80 conflicts):
# In AdGuardHome.yaml, find the http section and set:
#   bind_host: 0.0.0.0
#   bind_port: 8081
```

If you don't have a backup, you can either:

- **Use the template:** `cp adguardhome/conf/AdGuardHome.yaml.example adguardhome/conf/AdGuardHome.yaml`,
  edit it (set your NextDNS config ID, generate a bcrypt password hash), and start.
- **Use the wizard:** AdGuard Home will start its first-run wizard on port 3000.
  Configure everything fresh through the web UI.

### 4.3 Start AdGuard Home

```bash
docker compose up -d adguardhome
docker compose logs -f adguardhome
# Wait for "AdGuard Home is available at ..." then Ctrl+C
```

**First-run (no backup):** Open `http://<pi5-ip>:3000` and complete the wizard.
Set the admin UI to port 8081 during setup.

**With restored config:** Open `http://<pi5-ip>:8081` (or whatever port was in your
`AdGuardHome.yaml`).

Validate DNS resolution from another machine:

```bash
dig @<pi5-ip> example.com
```

### 4.4 Point Firewalla DNS at the Pi 5

Once you've confirmed AdGuard Home is resolving queries:

1. Open the Firewalla app.
2. Go to **Network → DNS** (or equivalent for your Firewalla model).
3. Change the DNS servers from NextDNS to the Pi 5's IP address.
4. Optionally keep NextDNS as a secondary/fallback in Firewalla.

Configure NextDNS as the **upstream** inside AdGuard Home (Settings → DNS Settings →
Upstream DNS servers) to layer AdGuard's local filtering with NextDNS's cloud
filtering.

### 4.5 Start MongoDB + UniFi Network Application

```bash
docker compose up -d unifi-db
# Wait ~30 seconds for MongoDB to initialize and run init-mongo.sh
docker compose logs unifi-db | tail -20
# Confirm you see "Successfully added user" or similar

docker compose up -d unifi-network-application
docker compose logs -f unifi-network-application
# Wait for startup to complete (~2 minutes on first run), then Ctrl+C
```

### 4.6 Restore UniFi backup

1. Open `https://<pi5-ip>:8443` in your browser (accept the self-signed cert warning).
2. The first-run wizard will appear. Choose **Restore from a previous backup**.
3. Upload the `.unf` file from Phase 1.2 (`unifi/backups/` in this repo).
4. Wait for the restore to complete and the controller to restart (~2-5 minutes).
5. Log in with your existing UniFi credentials.

### 4.7 AP re-adoption

**If you reused the N150's IP (recommended):**

APs should reconnect automatically within a few minutes. Check
**Devices** in the UniFi web UI — they should appear as "Connected" one by one.

**If you used a different IP:**

SSH into each AP and manually set the new inform URL:

```bash
ssh ubnt@<ap-ip>
set-inform http://<pi5-ip>:8080/inform
# Run it twice — UniFi sometimes needs the second attempt
set-inform http://<pi5-ip>:8080/inform
```

Default AP credentials: username `ubnt`, password `ubnt` (unless changed).

Also update the inform host in the UniFi UI: **Settings → System → Advanced →
Inform Host** — set it to the Pi 5's IP and check "Override".

---

## Phase 5 — Smoke tests

Run through this checklist after cutover:

- [ ] `dig @<pi5-ip> example.com` returns a valid answer from multiple machines
- [ ] AdGuard Home query log (`http://<pi5-ip>:8081`) shows incoming queries
- [ ] AdGuard upstream is NextDNS and queries are flowing to it
- [ ] `https://<pi5-ip>:8443` loads the UniFi dashboard
- [ ] All APs show "Connected" in UniFi Devices
- [ ] Client count in UniFi matches expectations
- [ ] Wi-Fi clients can connect and get internet access
- [ ] Wired clients can resolve DNS
- [ ] Historical data (stats, events) from the UniFi backup is visible
- [ ] You can SSH into the Pi 5 from your Mac (confirms no VLAN/routing issues)

---

## Phase 6 — Day-2 operations

### Backups

UniFi controller state lives in MongoDB, not in `unifi/config/` alone. The backup
script uses `mongodump` to capture a consistent database snapshot alongside the
file-level config archives.

UniFi also generates its own `.unf` autobackup files inside `unifi/config/data/backup/autobackup/`.
Enable autobackups in the UniFi UI (**Settings → System → Backups → Auto Backup**)
and keep the default retention. The nightly script archives these as well.

```bash
# Create the backup script
cat << 'SCRIPT' | sudo tee /opt/docker-pi/backup.sh
#!/bin/bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
BACKUP_DIR="$SCRIPT_DIR/backups"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
RETENTION_DAYS=14

# Source .env for MongoDB credentials
set -a
source "$SCRIPT_DIR/.env"
set +a

mkdir -p "$BACKUP_DIR"

# --- AdGuard Home (config + runtime data: stats, query log, filters) ---
tar czf "$BACKUP_DIR/adguard-$TIMESTAMP.tar.gz" \
  -C "$SCRIPT_DIR" adguardhome/conf/ adguardhome/work/

# --- UniFi: MongoDB dump (only the three UniFi databases) ---
docker exec unifi-db mongodump \
  --authenticationDatabase="$MONGO_AUTHSOURCE" \
  -u "$MONGO_INITDB_ROOT_USERNAME" \
  -p "$MONGO_INITDB_ROOT_PASSWORD" \
  --nsInclude="${MONGO_DBNAME}.*" \
  --nsInclude="${MONGO_DBNAME}_stat.*" \
  --nsInclude="${MONGO_DBNAME}_audit.*" \
  --archive --gzip \
  > "$BACKUP_DIR/mongo-$TIMESTAMP.archive.gz"

# --- UniFi: config dir (certs, system.properties, autobackup .unf files) ---
tar czf "$BACKUP_DIR/unifi-config-$TIMESTAMP.tar.gz" \
  -C "$SCRIPT_DIR" unifi/config/

# --- Prune old backups ---
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "*.archive.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $TIMESTAMP"
SCRIPT

sudo chmod +x /opt/docker-pi/backup.sh

# Add to cron (runs at 3:00 AM daily)
(crontab -l 2>/dev/null; echo "0 3 * * * /opt/docker-pi/backup.sh >> /var/log/docker-pi-backup.log 2>&1") | crontab -
```

**What each backup artifact covers:**

| File                                  | Contents                                          |
|---------------------------------------|---------------------------------------------------|
| `adguard-<ts>.tar.gz`                 | AdGuardHome.yaml, downloaded filter snapshots, stats.db, query log |
| `mongo-<ts>.archive.gz`              | UniFi MongoDB databases only: `unifi`, `unifi_stat`, `unifi_audit` |
| `unifi-config-<ts>.tar.gz`           | system.properties, certs, autobackup `.unf` files |

For a complete UniFi restore you need **both** the MongoDB dump and the config dir
(or just a recent `.unf` file from the autobackup directory — see Recovery below).

Optionally `rsync` the backup directory to another host (N150, NAS, etc.) for
off-device redundancy.

### Upgrades

1. Check the release notes for the new image version.
2. For UniFi upgrades, **always download a backup from the web UI first**
   (Settings → System → Backups → Download Backup).
3. Edit `docker-compose.yml` and bump the image tag.
4. Pull and restart:

```bash
cd /opt/docker-pi
docker compose pull
docker compose up -d
docker compose logs -f    # watch for errors
```

5. Commit the tag change to git:

```bash
git add docker-compose.yml
git commit -m "upgrade: <service> to <version>"
git push
```

### Recovery test

Periodically verify that a full restore works. There are two restore paths:

**Option A — Restore from `.unf` file (simpler, recommended):**

1. Stop the stack: `docker compose down`
2. Move data aside: `mv unifi/config unifi/config.bak && sudo mv mongo/data mongo/data.bak`
3. Start fresh: `docker compose up -d`
4. Open `https://<pi5-ip>:8443`, choose **Restore from a previous backup** in the wizard.
5. Upload a `.unf` file (from `backups/` or from `unifi/config.bak/data/backup/autobackup/`).
6. Confirm APs reconnect, data is present, DNS still works.
7. Clean up: `rm -rf unifi/config.bak && sudo rm -rf mongo/data.bak`

**Option B — Restore from `mongodump` archive (preserves full DB state):**

1. Stop the stack: `docker compose down`
2. Move data aside: `mv unifi/config unifi/config.bak && sudo mv mongo/data mongo/data.bak`
3. Start only MongoDB: `docker compose up -d unifi-db`
4. Wait ~15 seconds for MongoDB to initialize, then restore:

```bash
# Source credentials from .env
set -a; source .env; set +a

docker exec -i unifi-db mongorestore \
  --authenticationDatabase="$MONGO_AUTHSOURCE" \
  -u "$MONGO_INITDB_ROOT_USERNAME" \
  -p "$MONGO_INITDB_ROOT_PASSWORD" \
  --drop \
  --archive --gzip \
  < backups/mongo-<TIMESTAMP>.archive.gz
```

5. Restore the config directory: `tar xzf backups/unifi-config-<TIMESTAMP>.tar.gz -C .`
6. Start UniFi: `docker compose up -d unifi-network-application`
7. Confirm APs reconnect, data is present, DNS still works.
8. Clean up: `rm -rf unifi/config.bak && sudo rm -rf mongo/data.bak`

### Container health checks

```bash
# Quick status
docker compose ps

# Resource usage
docker stats --no-stream

# Service logs
docker compose logs adguardhome --tail 50
docker compose logs unifi-db --tail 50
docker compose logs unifi-network-application --tail 50
```

### Decommissioning old hardware

Once the Pi 5 is stable and verified:

- **N150 mini-PC:** Power off. Keep the `.unf` backup file saved elsewhere. The N150
  can serve as a cold spare or be repurposed.
- **Pi 4:** Power off. Keep the SD card as a backup of the old AdGuard config.
  The Pi 4 can be repurposed or kept as a cold spare.

---

## File layout

```
docker-pi/
├── README.md                    # This file
├── docker-compose.yml           # All three services
├── .env.example                 # Template — copy to .env and fill in secrets
├── init-mongo.sh                # MongoDB init script (creates unifi DB user)
├── adguardhome/
│   ├── conf/
│   │   └── AdGuardHome.yaml.example  # Sanitized config template (committed)
│   │                                  # AdGuardHome.yaml is gitignored (has secrets)
│   └── work/                    # Runtime data (gitignored)
├── unifi/
│   ├── config/                  # UniFi app data + MongoDB (gitignored)
│   └── backups/                 # Place .unf files here for restore
├── mongo/
│   └── data/                    # MongoDB data files (gitignored)
└── docs/
    ├── decisions.md             # Architecture decision records
    └── ansible-future.md        # Notes for Ansible conversion
```

---

## Ports reference

| Port       | Protocol | Service       | Purpose                     |
|------------|----------|---------------|-----------------------------|
| 53         | TCP/UDP  | AdGuard Home  | DNS                         |
| 3000       | TCP      | AdGuard Home  | First-run wizard only       |
| 8081       | TCP      | AdGuard Home  | Admin UI (post-setup)       |
| 8443       | TCP      | UniFi         | Web admin (HTTPS)           |
| 8080       | TCP      | UniFi         | Device inform               |
| 3478       | UDP      | UniFi         | STUN                        |
| 10001      | UDP      | UniFi         | AP discovery                |
| 1900       | UDP      | UniFi         | L2 discovery (optional)     |
| 8843       | TCP      | UniFi         | HTTPS portal (optional)     |
| 8880       | TCP      | UniFi         | HTTP portal (optional)      |
| 6789       | TCP      | UniFi         | Speed test (optional)       |
