# docker-pi

Docker Compose stack for a Raspberry Pi running **AdGuard Home** (DNS sinkhole) and
**UniFi Network Application** (wireless controller).

## Architecture

```
LAN clients ──DNS:53──▶ Router ──forward──▶ Pi (AdGuard Home) ──DoH/DoT──▶ Upstream DNS
UniFi APs ──inform:8080──▶ Pi (UniFi Network Application) ──▶ MongoDB (internal)
```

- **AdGuard Home** runs with `network_mode: host` so it owns port 53 directly.
- **UniFi Network Application** and **MongoDB** share a Docker bridge network
  (`unifi-net`). UniFi ports are mapped to the host explicitly.
- MongoDB is not exposed outside Docker — only reachable by the UniFi container.

See [docs/decisions.md](docs/decisions.md) for full rationale.

## Deployment modes

The UniFi services are gated behind a Docker Compose **profile**, so you can run
AdGuard Home by itself or the full stack:

| Mode                | Command                              | Services started                       |
|---------------------|--------------------------------------|----------------------------------------|
| AdGuard only (default) | `docker compose up -d`            | `adguardhome`                          |
| Full stack          | `docker compose --profile unifi up -d` | `adguardhome`, `unifi-db`, `unifi-network-application` |

If you only need DNS for now, use the default. The UniFi Phases below (1.2, 4.5–4.7)
are optional — skip them unless you're running the controller here. To add UniFi
later, just start the stack with `--profile unifi` and follow those phases.

---

## Prerequisites

- Raspberry Pi (tested on Pi 5; Pi 4 with 4GB+ should also work)
- Storage: NVMe drive, SD card, or both (see Phase 2 for options)
- Ethernet connection to LAN
- Static IP reserved in your router's DHCP settings
- SSH access to the Pi from your workstation

---

## Phase 1 — Back up existing services (migration only)

Skip this phase if you're doing a fresh install with no prior AdGuard or UniFi setup.

### 1.1 Back up AdGuard Home

SSH into the machine currently running AdGuard Home and create a tarball:

```bash
# Native install (adjust the path to wherever AdGuard Home lives):
sudo tar czf /tmp/adguard-backup.tar.gz -C /home/<user> AdGuardHome/
# or: sudo tar czf /tmp/adguard-backup.tar.gz -C /opt AdGuardHome/

# Docker install — find the bound volume first:
docker inspect adguardhome | grep -A5 Mounts
# Then tar the conf and work directories.
```

Copy the tarball to your workstation:

```bash
scp <user>@<old-host>:/tmp/adguard-backup.tar.gz ~/
```

**If SSH is unavailable**, power off the machine and mount its storage (SD card, SSD)
on another computer to copy the files directly.

**What you need from the backup:**

| File / directory                     | Purpose                                  |
|--------------------------------------|------------------------------------------|
| `AdGuardHome.yaml`                   | All settings: upstreams, filters, rewrites, clients |
| `data/filters/`                      | Downloaded blocklist snapshots            |
| `data/stats.db`                      | Query statistics                         |
| `data/querylog.json` (or `.jsonl`)   | Query log history                        |
| `data/sessions.db`                   | Login sessions (can be discarded)        |

### 1.2 Back up / migrate UniFi Network Application

There are two approaches depending on your source setup.

**If migrating from a UniFi OS device (Cloud Key, UDM, or UniFi OS Server on PC):**

The OS-level backup ("Backup for All Applications and Settings") **cannot** be restored
into a standalone Docker container. Use the **Export Site** migration wizard instead:

1. Open the Network application on the source device.
2. Go to **Settings → System → Site Management → Export Site**.
3. Click **Export Site**, then **Download the Site Export File** (save it — this is
   your backup of record).
4. Click **Continue**. Enter the new controller's IP and port **8080** (the inform
   port, not 8443).
5. Select all devices, then click **Migrate Devices**. This pushes the site config
   and re-adopts all devices to the new controller in one step.

Note the **Network Application version** on the source (shown at the bottom-left of
the Network app UI). The destination Docker image must be equal to or newer.

**If migrating from a standalone UniFi Network Application (Docker, apt, etc.):**

1. Navigate to **Settings → System → Backups**.
2. Click **Download Backup** and save the `.unf` file.
3. Note the Network Application version.

Place backup files in `unifi/backups/` in this repo (gitignored).

---

## Phase 2 — Pi setup

### 2.1 Choose a boot configuration

**Option A — Boot from NVMe (simpler):**

Use **Raspberry Pi Imager** to write **Raspberry Pi OS Lite (64-bit)** directly to
the NVMe (via USB enclosure or by booting from a temporary SD card first). In Imager's
advanced settings (gear icon), pre-configure hostname, SSH, username/password, locale,
and optionally Wi-Fi.

**Option B — Boot from SD card, use NVMe as data drive (recommended if NVMe boot is
unreliable):**

Image a microSD card with Raspberry Pi OS Lite (64-bit) using the same Imager
settings. Boot from the SD card, then update the OS and format/mount the NVMe for
Docker data:

```bash
# Update the OS and firmware first — keeps rpi-eeprom current and avoids
# bootloader/EEPROM quirks during the Argon and boot-order steps below
sudo apt update && sudo apt full-upgrade -y

# Confirm the NVMe is detected
lsblk

# Wipe and create a single ext4 partition (this erases the entire NVMe)
sudo parted /dev/nvme0n1 mklabel gpt
sudo parted /dev/nvme0n1 mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/nvme0n1p1

# Mount and persist across reboots
sudo mkdir -p /mnt/nvme
sudo mount /dev/nvme0n1p1 /mnt/nvme
# NOTE: nofail is critical. Without it, if the NVMe isn't detected at boot
# (Pi 5 PCIe/NVMe detection can be flaky), systemd blocks boot waiting for the
# mount and drops to an emergency shell — the Pi appears dead and unreachable.
echo '/dev/nvme0n1p1 /mnt/nvme ext4 defaults,noatime,nofail,x-systemd.device-timeout=10 0 2' | sudo tee -a /etc/fstab
sudo chown -R $USER:$USER /mnt/nvme

# ALWAYS validate fstab before rebooting — a bad entry is itself a boot-breaker
sudo umount /mnt/nvme && sudo mount -a && mount | grep nvme

# Reload systemd so it picks up the new fstab entry (clears the "fstab modified" hint)
sudo systemctl daemon-reload
```

If NVMe detection is intermittent at boot (you sometimes see `NVME off` on the
bootloader screen, or the Pi randomly fails to come back after a reboot), this is a
known Pi 5 + third-party M.2 HAT signal-integrity issue. Force a slower PCIe link
speed by adding to `/boot/firmware/config.txt`:

```
dtparam=pciex1
dtparam=pciex1_gen=2
```

Drop to `_gen=1` if Gen 2 is still unstable. Also reseat the FPC ribbon cable on
both ends.

### 2.2 NVMe boot order (Option A only)

If the Pi boots successfully from NVMe, skip this. You only need it if the Pi
**won't boot from NVMe** and you had to fall back to an SD card.

SSH in and check:

```bash
sudo rpi-eeprom-config
```

Look for `BOOT_ORDER` — it should contain `6` (NVMe). Default Pi 5 EEPROMs already
prefer NVMe. If it's missing:

```bash
sudo rpi-eeprom-config --edit
# Change or add: BOOT_ORDER=0xf416
# Save and exit, then reboot:
sudo reboot
```

### 2.3 Argon NEO 5 case fan driver (if applicable)

If you're using an **Argon NEO 5 M.2 NVMe case**, install the fan control driver.
Without it, the fan runs at full speed.

```bash
curl https://download.argon40.com/argon-eeprom.sh | bash
sudo reboot
```

After reboot:

```bash
curl https://download.argon40.com/argonneo5.sh | bash
sudo reboot
```

After the second reboot, confirm the fan is being managed (not stuck at full speed):

```bash
vcgencmd measure_temp
```

A low idle temp with a quiet fan means it's working. (Note: the `argonone-config`
tuning CLI is not always installed for the NEO 5 variant — the default fan curve is
fine. Run `ls /usr/bin | grep -i argon` to see what control tools you have.)

### 2.4 Auto-boot after power loss

The Pi 5 **auto-boots after a power outage by default** — no setting is required for
this. The only thing that breaks it is `WAIT_FOR_POWER_BUTTON=1` together with
`POWER_OFF_ON_HALT=1`, which forces a physical button press on the first power-up after
the cable is reconnected.

If your Pi requires a button press after an outage, check the EEPROM config:

```bash
rpi-eeprom-config | grep -E 'POWER_OFF_ON_HALT|WAIT_FOR_POWER_BUTTON|WAKE_ON_GPIO'
```

Ensure `WAIT_FOR_POWER_BUTTON` is `0` or absent. To change it:

```bash
sudo rpi-eeprom-config --edit
# Set (or remove the line entirely):
#   WAIT_FOR_POWER_BUTTON=0
# Save and exit, then reboot.
```

Notes:
- `POWER_OFF_ON_HALT=1` (set by the Argon fan driver for low-power halt + its power
  button) is fine and still auto-boots after an outage.
- `WAKE_ON_GPIO` has no effect on the Pi 5 (it uses the dedicated power button, not
  GPIO3).
- If the Pi instead fails to come back after a *reboot* (not a power loss), that's a
  boot hang, not a power setting — see the NVMe `nofail` note in Phase 2.1.

Verify by actually cutting power at the wall/strip, waiting ~10s, and restoring it —
the Pi should boot on its own without a button press.

### 2.5 Static IP

Reserve the Pi's MAC address with a static IP in your router's DHCP settings.

If migrating from an existing UniFi controller, **reuse the old controller's IP** so
APs re-adopt automatically.

---

## Phase 3 — Host bootstrap

Run these on the Pi over SSH. Each section is a discrete concern, designed to map
cleanly to an Ansible role later (see [docs/ansible-future.md](docs/ansible-future.md)).

### 3.1 System update and timezone

```bash
sudo apt update && sudo apt full-upgrade -y
sudo timedatectl set-timezone <YOUR_TIMEZONE>   # e.g. America/New_York
sudo apt install -y git unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades  # enable auto security updates
```

### 3.2 Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
exit
```

Log back in, then verify:

```bash
docker --version
docker compose version
```

**If using the SD + NVMe setup (Option B)**, move Docker's data root to the NVMe so
container images and volumes don't wear out the SD card:

```bash
sudo systemctl stop docker
sudo mkdir -p /mnt/nvme/docker-data
echo '{"data-root": "/mnt/nvme/docker-data"}' | sudo tee /etc/docker/daemon.json
sudo systemctl start docker

# Verify
docker info | grep "Docker Root Dir"
# Should show: /mnt/nvme/docker-data
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

**NVMe-only boot (Option A):**

```bash
git clone https://github.com/<your-username>/docker-pi.git ~/docker-pi
cd ~/docker-pi
```

**SD + NVMe (Option B):**

```bash
git clone https://github.com/<your-username>/docker-pi.git /mnt/nvme/docker-pi
cd /mnt/nvme/docker-pi
```

All subsequent commands assume you're in the repo directory.

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
MONGO_INITDB_ROOT_PASSWORD=CHANGEME_root_password    <- paste first password
MONGO_PASS=CHANGEME_unifi_password                   <- paste second password
```

The other defaults should be fine as-is:

- `PUID`/`PGID`: should match your user. Run `id` — if you see `uid=1000` and
  `gid=1000`, no change needed.
- `TZ`: set to your timezone (e.g. `America/New_York`).

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

### 4.2 Restore AdGuard Home config (optional)

If you have the backup tarball from Phase 1.1, copy it from the machine that has it
to the Pi. Run this **from the machine with the backup** (not on the Pi):

```bash
scp adguard-backup.tar.gz <pi-user>@<pi-ip>:<repo-dir>/
```

Then **on the Pi**, extract and place the config:

```bash
tar xzf adguard-backup.tar.gz -C /tmp
```

The extracted path depends on how AdGuard was installed on the source machine. Find
`AdGuardHome.yaml` and copy it into the conf directory:

```bash
# Adjust the path to match your backup structure:
cp /tmp/AdGuardHome/AdGuardHome.yaml ./adguardhome/conf/
# or: cp /tmp/opt/AdGuardHome/AdGuardHome.yaml ./adguardhome/conf/
```

Edit the config to move the admin UI off port 80:

```bash
nano ./adguardhome/conf/AdGuardHome.yaml
```

Find the `http` section near the top and set the address to port 8081:

```yaml
http:
  address: 0.0.0.0:8081
```

Save and exit. Clean up the tarball: `rm adguard-backup.tar.gz`

**If you don't have a backup**, you have two options:

- **Use the template:** `cp adguardhome/conf/AdGuardHome.yaml.example adguardhome/conf/AdGuardHome.yaml`,
  edit it (set your upstream DNS, generate a bcrypt password hash), and start.
- **Use the wizard:** Skip this step entirely. AdGuard Home will start its first-run
  wizard on port 3000 where you can configure everything through the web UI.

### 4.3 Start AdGuard Home

```bash
docker compose up -d adguardhome
docker compose logs -f adguardhome
```

> **Note:** `Ctrl+C` here only stops the log tail — the container keeps running in
> the background. Wait until you see "AdGuard Home is available at ..." then press
> `Ctrl+C` to get your shell back.

**First-run (no backup):** Open `http://<pi-ip>:3000` and complete the wizard.
Set the admin UI to port 8081 during setup.

**With restored config:** Open `http://<pi-ip>:8081` (or whatever port was in your
`AdGuardHome.yaml`).

Validate DNS resolution from another machine on your network:

```bash
dig @<pi-ip> example.com
```

### 4.4 Point your router's DNS at the Pi

Once you've confirmed AdGuard Home is resolving queries, update your router's DNS
settings to forward to the Pi's IP address.

If you use an upstream filtering service (e.g. NextDNS, Cloudflare Gateway), configure
it as the **upstream** inside AdGuard Home (Settings → DNS Settings → Upstream DNS
servers) to layer AdGuard's local filtering with the cloud service.

### 4.5 Start MongoDB + UniFi Network Application

> **Optional — skip if you're running AdGuard Home only.** The UniFi services live
> behind the `unifi` Compose profile, so they need the `--profile unifi` flag.

```bash
docker compose --profile unifi up -d unifi-db
```

Wait ~30 seconds for MongoDB to initialize and run the init script:

```bash
docker compose logs unifi-db | tail -20
```

Confirm you see "Successfully added user" or similar, then start UniFi:

```bash
docker compose --profile unifi up -d unifi-network-application
docker compose logs -f unifi-network-application
```

Tip: `docker compose --profile unifi up -d` starts the whole stack (AdGuard + both
UniFi services) in one command.

> **Note:** `Ctrl+C` here only stops the log tail — the container keeps running in
> the background. Wait for startup to complete (~2 minutes on first run), then press
> `Ctrl+C` to get your shell back.

### 4.6 Migrate or restore UniFi

**If you used the Export Site wizard (from UniFi OS):**

1. Open `https://<pi-ip>:8443` and complete the initial setup wizard (create a local
   admin account, skip cloud login if you prefer).
2. The migration wizard from Phase 1.2 will push the site config and devices
   automatically. Check **Devices** in the web UI — they should appear as "Connected"
   within a few minutes.
3. If the migration wizard was already completed during Phase 1.2, devices should
   already be adopting.

**If you have a `.unf` backup file (from a standalone controller):**

1. Open `https://<pi-ip>:8443` in your browser (accept the self-signed cert warning).
2. The first-run wizard will appear. Choose **Restore from a previous backup**.
3. Upload the `.unf` file from Phase 1.2 (`unifi/backups/` in this repo).
4. Wait for the restore to complete and the controller to restart (~2-5 minutes).
5. Log in with your existing UniFi credentials.

### 4.7 AP re-adoption (if devices didn't migrate automatically)

**If you reused the old controller's IP:**

APs should reconnect automatically within a few minutes. Check **Devices** in the
UniFi web UI — they should appear as "Connected" one by one.

**If you used a different IP:**

SSH into each AP and manually set the new inform URL:

```bash
ssh ubnt@<ap-ip>
set-inform http://<pi-ip>:8080/inform
set-inform http://<pi-ip>:8080/inform
```

Default AP credentials: username `ubnt`, password `ubnt` (unless changed).

Also update the inform host in the UniFi UI: **Settings → System → Advanced →
Inform Host** — set it to the Pi's IP and check "Override".

---

## Phase 5 — Smoke tests

Run through this checklist after cutover:

- [ ] `dig @<pi-ip> example.com` returns a valid answer from multiple machines
- [ ] AdGuard Home query log (`http://<pi-ip>:8081`) shows incoming queries
- [ ] AdGuard upstream DNS is configured and queries are flowing
- [ ] `https://<pi-ip>:8443` loads the UniFi dashboard
- [ ] All APs show "Connected" in UniFi Devices
- [ ] Client count in UniFi matches expectations
- [ ] Wi-Fi clients can connect and get internet access
- [ ] Wired clients can resolve DNS
- [ ] Historical data (stats, events) from the UniFi backup is visible
- [ ] You can SSH into the Pi from your workstation

---

## Phase 6 — Day-2 operations

### Backups

UniFi controller state lives in MongoDB, not in `unifi/config/` alone. The backup
script uses `mongodump` to capture a consistent database snapshot alongside the
file-level config archives.

UniFi also generates its own `.unf` autobackup files inside
`unifi/config/data/backup/autobackup/`. Enable autobackups in the UniFi UI
(**Settings → System → Backups → Auto Backup**) and keep the default retention.

```bash
cat << 'SCRIPT' | sudo tee <repo-dir>/backup.sh
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

# --- UniFi (only if the UniFi stack is running) ---
if docker ps --format '{{.Names}}' | grep -q '^unifi-db$'; then
  # MongoDB dump (only the three UniFi databases)
  docker exec unifi-db mongodump \
    --authenticationDatabase="$MONGO_AUTHSOURCE" \
    -u "$MONGO_INITDB_ROOT_USERNAME" \
    -p "$MONGO_INITDB_ROOT_PASSWORD" \
    --nsInclude="${MONGO_DBNAME}.*" \
    --nsInclude="${MONGO_DBNAME}_stat.*" \
    --nsInclude="${MONGO_DBNAME}_audit.*" \
    --archive --gzip \
    > "$BACKUP_DIR/mongo-$TIMESTAMP.archive.gz"

  # config dir (certs, system.properties, autobackup .unf files)
  tar czf "$BACKUP_DIR/unifi-config-$TIMESTAMP.tar.gz" \
    -C "$SCRIPT_DIR" unifi/config/
fi

# --- Prune old backups ---
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "*.archive.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $TIMESTAMP"
SCRIPT

sudo chmod +x <repo-dir>/backup.sh

# Add to cron (runs at 3:00 AM daily) — adjust the path to match your repo location
(crontab -l 2>/dev/null; echo "0 3 * * * <repo-dir>/backup.sh >> /var/log/docker-pi-backup.log 2>&1") | crontab -
```

Replace `<repo-dir>` with the absolute path to your repo (e.g. `/mnt/nvme/docker-pi`
or `~/docker-pi`).

**What each backup artifact covers:**

| File                                  | Contents                                          |
|---------------------------------------|---------------------------------------------------|
| `adguard-<ts>.tar.gz`                 | AdGuardHome.yaml, downloaded filter snapshots, stats.db, query log |
| `mongo-<ts>.archive.gz`              | UniFi MongoDB databases only: `unifi`, `unifi_stat`, `unifi_audit` |
| `unifi-config-<ts>.tar.gz`           | system.properties, certs, autobackup `.unf` files |

For a complete UniFi restore you need **both** the MongoDB dump and the config dir
(or just a recent `.unf` file from the autobackup directory — see Recovery below).

Optionally `rsync` the backup directory to another host for off-device redundancy.

### Upgrades

1. Check the release notes for the new image version.
2. For UniFi upgrades, **always download a backup from the web UI first**
   (Settings → System → Backups → Download Backup).
3. Edit `docker-compose.yml` and bump the image tag.
4. Pull and restart:

```bash
# AdGuard-only deployment:
docker compose pull
docker compose up -d

# Full stack (include UniFi services):
docker compose --profile unifi pull
docker compose --profile unifi up -d

docker compose logs -f    # watch for errors
```

5. Commit the tag change to git:

```bash
git add docker-compose.yml
git commit -m "upgrade: <service> to <version>"
git push
```

### Recovery test

Periodically verify that a full restore works (UniFi only — requires the `unifi`
profile). There are two restore paths:

**Option A — Restore from `.unf` file (simpler, recommended):**

1. Stop the stack: `docker compose --profile unifi down`
2. Move data aside: `mv unifi/config unifi/config.bak && sudo mv mongo/data mongo/data.bak`
3. Start fresh: `docker compose --profile unifi up -d`
4. Open `https://<pi-ip>:8443`, choose **Restore from a previous backup** in the wizard.
5. Upload a `.unf` file (from `backups/` or from `unifi/config.bak/data/backup/autobackup/`).
6. Confirm APs reconnect, data is present, DNS still works.
7. Clean up: `rm -rf unifi/config.bak && sudo rm -rf mongo/data.bak`

**Option B — Restore from `mongodump` archive (preserves full DB state):**

1. Stop the stack: `docker compose --profile unifi down`
2. Move data aside: `mv unifi/config unifi/config.bak && sudo mv mongo/data mongo/data.bak`
3. Start only MongoDB: `docker compose --profile unifi up -d unifi-db`
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
6. Start UniFi: `docker compose --profile unifi up -d unifi-network-application`
7. Confirm APs reconnect, data is present, DNS still works.
8. Clean up: `rm -rf unifi/config.bak && sudo rm -rf mongo/data.bak`

### Container health checks

```bash
docker compose ps                # quick status
docker stats --no-stream         # resource usage
docker compose logs <service> --tail 50
```

---

## Troubleshooting

### AdGuard Home crash-loops with "address already in use"

```
panic: listen tcp 0.0.0.0:80: bind: address already in use
```

Something else on the host already owns the port AdGuard wants (port 80 if you set the
admin UI to 80, or port 53 for DNS). Find the offender and stop it:

```bash
sudo ss -tlnp | grep -E ':80 |:53 '
```

Common culprits:
- **apache2 / nginx** on port 80 — often installed accidentally (e.g. just to get
  `htpasswd` for a password hash). Stop and disable it: `sudo systemctl disable --now apache2`.
  You don't need a full web server for a bcrypt hash — AdGuard's first-run wizard makes
  one, or use `docker run --rm httpd:alpine htpasswd -nbB admin 'password'`.
- **systemd-resolved** on port 53 — see Phase 3.3.

### NVMe not detected / drops out (Pi 5 + M.2 HAT)

If `lsblk` is missing `nvme0n1`, or `dmesg` shows `nvme nvme0: Device not ready;
aborting initialisation, CSTS=0x0`, the PCIe link is too aggressive for the HAT.
Check the negotiated speed:

```bash
dmesg | grep 'link up'    # 8.0 GT/s = Gen 3 (unstable on most HATs); 5.0 = Gen 2
```

Force a slower, stable speed in `/boot/firmware/config.txt` (`dtparam=pciex1_gen=2`,
or `=1` if Gen 2 still drops), then reboot. See Phase 2.1. The `nofail` fstab option
ensures the Pi still boots and stays reachable even when the NVMe is absent.

### Data silently landing on the SD card instead of the NVMe

If the NVMe fails to mount, the `/mnt/nvme` path falls through to the SD card and
writes go there silently. Verify where data actually lives:

```bash
findmnt /mnt/nvme    # SOURCE must be /dev/nvme0n1p1, not /dev/mmcblk0p2
```

The `RequiresMountsFor=/mnt/nvme` Docker drop-in (Phase 4) prevents this by refusing
to start Docker unless the NVMe is mounted.

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
│   ├── config/                  # UniFi app data (gitignored)
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
