# PiNAS

Self-hosted NAS on a Raspberry Pi 4B. Replaces an end-of-life network-attached storage device with a maintainable, documented alternative using commodity hardware and open-source software.

---

## Architecture

```
Computer
  |
  | rsync over SSH (daily, anacron)
  |
Router
  |
Raspberry Pi 4B (Debian Trixie ARM64)
  |
  +-- Powered USB Hub
        |
        +-- NVMe Enclosure --> 1TB NVMe SSD
        +-- NVMe Enclosure --> 1TB NVMe SSD
              |
              mdadm RAID 1
                    |
                    ext4 filesystem
                          |
                          /mnt/nas
                                |
                                FileBrowser Quantum (LAN only)
```

---

## Stack

| Layer | Component |
|---|---|
| OS | Debian Trixie ARM64 |
| RAID | mdadm RAID 1 (software mirror) |
| Filesystem | ext4 |
| File manager | FileBrowser Quantum |
| Backup | rsync over SSH via anacron |
| Service management | systemd |

---

## Design Decisions

### What was evaluated and rejected

**Raspberry Pi 5** — PCIe NVMe support via HAT eliminates the USB power budget constraint entirely. Ended up not selection because the Pi 4B was already in service and a powered hub resolves the power limitation at lower cost. Pi 5 remains the cleaner hardware choice for a greenfield build.

**OpenMediaVault** — Designed for clean OS installs. Would conflict with existing Nginx, Certbot, and cron configuration already running on the Pi for another project. Rejected in favour of composing individual tools.

**Samba** — Initially considered for Time Machine support. Dropped when Time Machine was dropped; it requires AFP emulation via the `fruit` module and is all-or-nothing. rsync handles the actual backup requirement with less complexity.

**Time Machine** — Cannot be scoped to a single folder. It snapshots the full system state regardless of exclusions. rsync is selective by design.

**Public subdomain with TLS** — Would require ongoing certificate management, authentication layer, and permanent attack surface exposure. The use case (storage, home network) doesn't justify it. LAN-only access with SSH tunnel on demand covers the rare exception.

**WireGuard VPN** — The correct long-term solution for remote access, but deferred. One-time setup cost, near-zero maintenance after.

**Filebrowser (original)** — v2.63.4 has an unresolved routing bug (`e.params.catchAll is not iterable`). Switched to FileBrowser Quantum, the actively maintained fork.

**cron** — Skips missed jobs. If the computer is asleep or the connection drops, the backup simply doesn't run. anacron runs missed jobs on next availability; correct behaviour for a backup that doesn't need to run at a specific clock time.

---

## Hardware Constraint: Pi 4B USB Power Budget

The Pi 4B has a 1.2A shared budget across all USB-A ports. Two bus-powered NVMe enclosures exceed this on spin-up; confirmed via `dmesg`.

Resolution: A powered USB hub with independent PSU. Drives draw from the hub PSU, Pi sees normal USB traffic.

This is a known limitation when using the Pi 4B outside its original design intent. It's not a flaw, it's a constraint to design around.

---

## Backup Scope

Daily incremental rsync of selected folders from connected devices to the NAS. Managed via anacron; runs daily, tolerates missed runs. Designed to expand to additional devices over time.

---

## Security Posture

- **Least privilege** — FileBrowser Quantum runs as a dedicated system user with no login shell
- **LAN only** — no public exposure, no additional open ports
- **SSH key auth** — passwordless SSH uses ed25519 key pair, no password auth
- **Config-based credentials** — admin password set in `config.yaml`, survives database resets
- **CVE response** — updated to v1.3.3-stable same session as release (path traversal in public shares, GHSA-qqqm-5547-774x)

---

## Operational Reliability

- `nofail` flag in `/etc/fstab` — Pi boots normally if array fails to mount
- systemd `Restart=on-failure` — FileBrowser Quantum restarts automatically on crash
- mdadm write-intent bitmap — on power loss, only changed regions resync rather than the full array
- UUID-based fstab entry — array device name is stable across reboots
- tmux — long-running rsync sessions survive SSH disconnects

---

## Configuration

### `/opt/filebrowser/config.yaml`

```yaml
server:
  port: 8080
  cacheDir: /opt/filebrowser/tmp
  database: /opt/filebrowser/database.db
  sources:
    - path: "/mnt/nas"
      config:
        defaultEnabled: true
auth:
  adminUsername: "<stored in password manager>"
  adminPassword: "<stored in password manager>"
```

### `/etc/systemd/system/filebrowser.service`

```ini
[Unit]
Description=FileBrowser Quantum
After=network.target

[Service]
Type=simple
User=filebrowser
WorkingDirectory=/opt/filebrowser
ExecStart=/usr/local/bin/filebrowser -c /opt/filebrowser/config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

## Maintenance

```bash
# Array health
cat /proc/mdstat

# Service status
sudo systemctl status filebrowser

# Update FileBrowser Quantum
sudo systemctl stop filebrowser
curl -fsSL -o filebrowser https://github.com/gtsteffaniak/filebrowser/releases/latest/download/linux-arm64-filebrowser
chmod +x filebrowser
sudo mv filebrowser /usr/local/bin/filebrowser
sudo systemctl start filebrowser
```

---

## Related Projects

- [aersia-vip-player-android](https://github.com/AC-DAC/aersia-vip-player-android) — self-hosted music player, also running on this Pi
