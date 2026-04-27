# Project Roadmap

This document captures planned enhancements for the home-stash Raspberry Pi 5
server. Items are roughly ordered by priority.

---

## ✅ Implemented

- Raspberry Pi OS Lite base image
- Ansible playbooks for full unattended setup
- OpenMediaVault 7 with OMV-Extras
- Docker Engine via OMV-Extras compose plugin
- Immich self-hosted photo & video management
- Dependabot for automated dependency updates (GitHub Actions, Docker, Python)
- CodeQL security scanning for vulnerability detection
- Ansible Lint CI workflow for code quality enforcement
- Comprehensive Copilot instructions for secure, idempotent development
- External SSD storage for Immich (library and PostgreSQL database) via OpenMediaVault

---

## 🗂️ Backlog

### 0 - Miscellaneous automation things I need to do

* Fix ansible lint errors (CI workflow added, now need to address existing issues)
* Validate security posture (CodeQL scanning now active)
* Clean up old Immich data from SD card after confirming SSD migration works (see roadmap item #9)
* Implement local SD card backups for 3-2-1 strategy (see roadmap item #10)
* Think more about having procedures in place for data restore
* Probably should read more OMV and Immich docs to better understand the projects.... Oh AI, doing my thinking for me
* 

### 2 — Boot from SSD

**Goal:** Eliminate SD card as the primary boot device to improve reliability
and read/write throughput.

**Planned approach:**
- Use `rpi-eeprom-config` to set `BOOT_ORDER=0xf416` (USB boot priority)
- Copy the running OS image to the SSD using `rpi-clone` or `dd`
- Validate boot from SSD before removing the SD card
- Document the process in `docs/ssd-boot.md`

---

### 3 — Encrypted cloud backups

**Goal:** Automatically back up Immich data and the OMV configuration to cloud
storage (AWS S3 or Google Cloud Storage) with client-side encryption.

**Planned approach:**
- Use [Restic](https://restic.net/) for encrypted, deduplicated backups
- Support both AWS S3 and GCS backends via environment variable selection
- Ansible role that installs Restic, configures a `systemd` timer for nightly
  backups, and stores the repository password in a local secrets file
- Alert on backup failure via a simple e-mail or ntfy notification

---

### 4 — Full system restore from backup

**Goal:** Be able to recover the entire system to a new Pi from cloud backup
within 30 minutes in the event of hardware failure.

**Planned approach:**
- Document the restore runbook in `docs/restore.md`
- Provide an Ansible playbook (`playbooks/restore.yml`) that:
  1. Installs OMV and Docker on a fresh Pi
  2. Restores the Restic repository to the SSD
  3. Restarts all services and validates health

---

### 5 — Tailscale VPN

**Goal:** Access Immich, OMV, and any other services securely from outside the
home network without opening ports on the router.

**Planned approach:**
- Ansible role that installs Tailscale, authenticates with an auth key stored
  as an Ansible Vault secret, and enables the `tailscale` service
- Optionally advertise the Pi as a subnet router so all LAN devices are
  reachable over Tailscale
- Document the iOS/Android/desktop client setup in `docs/tailscale.md`

---

### 6 — Nextcloud (expanded storage)

**Goal:** Provide a broader file-sync and collaboration platform beyond photos
and videos.

**Planned approach:**
- Deploy Nextcloud as an additional Docker Compose stack
- Share the external SSD storage pool between Immich and Nextcloud
- Configure Nextcloud's external storage to also point at the Immich library
  (read-only) so files are browsable in both apps

---

### 7 — Enhanced local AI file management

**Goal:** On-device AI processing for smart albums, object recognition, and
automatic tagging without sending data to the cloud.

**Planned approach:**
- Immich already ships with a machine-learning container; tune it for Pi 5 ARM
- Explore [PhotoPrism](https://www.photoprism.app/) as an alternative with
  broader AI features
- Investigate connecting a Hailo-8L AI accelerator HAT for the Pi 5 to
  offload inference workloads

---

### 8 — K3s cluster across additional devices

**Goal:** Expand capacity and resilience by running a lightweight Kubernetes
cluster across the Pi 5 and any additional nodes (other Pis, old laptops, etc.)

**Planned approach:**
- Use [K3s](https://k3s.io/) — a lightweight Kubernetes distribution designed
  for ARM and resource-constrained nodes
- Ansible role that bootstraps the K3s server on the Pi 5 and joins additional
  agents
- Migrate Docker Compose workloads (Immich, Nextcloud) to Kubernetes manifests
  / Helm charts
- Add a shared storage layer (Longhorn or NFS) across nodes

---

### 9 — Clean up legacy SD card storage

**Goal:** Reclaim SD card space after validating the SSD migration is working
successfully.

**Planned approach:**
- Add a task to verify Immich is running successfully from SSD storage
- After validation period (7-14 days), remove old `/opt/immich/library` and
  `/opt/immich/postgres` directories from SD card
- Option to create a one-time backup before deletion
- Document the cleanup procedure in `docs/storage.md`

---

### 10 — Local SD card backups (3-2-1 strategy)

**Goal:** Implement local backups to SD card to move closer to the 3-2-1 backup
rule (3 copies, 2 different media types, 1 offsite).

**Current state:** 1 copy on SSD only

**Planned approach:**
- Use `rsync` to periodically backup Immich data from SSD to SD card
- Configure a `systemd` timer for nightly incremental backups
- Store backups in `/opt/immich-backups/` on SD card
- Implement retention policy (keep last 7 days, then weekly snapshots)
- Monitor backup health and send alerts on failure
- Combined with cloud backups (roadmap item #3), this achieves:
  - **3 copies:** SSD (primary), SD card (local), cloud (offsite)
  - **2 media:** SSD + SD card, cloud storage
  - **1 offsite:** Encrypted cloud backup

**Note:** This is complementary to cloud backups, not a replacement. SD card
provides fast local recovery, cloud provides disaster recovery.

---

### 11 — LUKS encryption for external storage

**Goal:** Add full-disk encryption to the external SSD for maximum data privacy
and security.

**Background:**
- LUKS provides AES-256 encryption for the entire partition
- Protects data if SSD is physically stolen or lost
- Pi 5 has hardware AES acceleration, so performance impact is minimal (5-15%)
- Requires passphrase/key to unlock the drive at mount time

**Security benefits:**
- Photos/videos are unreadable without the decryption key
- Industry-standard encryption (used by governments and enterprises)
- Open-source and audited implementation

**Trade-offs:**
- Adds complexity to setup and disaster recovery
- Requires key management (passphrase or key file)
- Cannot recover data if passphrase is lost
- May require manual unlock after reboot (unless key file is used)

**Planned approach:**
- Add `omv_encrypt_external_storage` variable to OMV role
- Store LUKS passphrase in Ansible Vault
- Create key file on SD card for automatic unlocking at boot
- Update storage setup tasks to:
  1. Create LUKS container with `cryptsetup luksFormat`
  2. Open container and format with ext4
  3. Configure `/etc/crypttab` for persistent unlocking
- Document encryption setup and key recovery procedures
- Provide option to encrypt existing drives (requires data migration)

**References:**
- [LUKS documentation](https://gitlab.com/cryptsetup/cryptsetup)
- [Raspberry Pi encryption guide](https://www.raspberrypi.com/documentation/computers/configuration.html#full-disk-encryption)

---

## Contributing

Have an idea not on this list? Open an issue or pull request!
