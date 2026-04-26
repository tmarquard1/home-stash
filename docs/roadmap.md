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

---

## 🗂️ Backlog

### 0 - Miscellaneous automation things I need to do

* Fix ansible lint errors (CI workflow added, now need to address existing issues)
* Validate security posture (CodeQL scanning now active)
* Think more about having procedures in place for data restore
* Probably should read more OMV and Immich docs to better understand the projects.... Oh AI, doing my thinking for me
* 

### 1 — External SSD for Immich storage

**Goal:** Move Immich's PostgreSQL database and media library off the SD card
onto a dedicated external SSD to improve performance and longevity.

**Hardware:**
- **USB hub:** Plugable 7-Port USB 3.0 Hub with 36 W power adapter — chosen
  for its reliable power delivery; the dedicated 36 W adapter ensures the hub
  can supply full USB 3.0 bus power to attached devices without back-powering
  or brown-outs on the Pi. The extra ports also leave room for future
  peripherals.
- **SSD:** Crucial X9 1 TB Portable SSD (USB 3.2 Gen 2, USB-C) — rated up to
  1050 MB/s sequential read. Connected through the Plugable hub, effective
  throughput is capped at USB 3.0 speeds (~5 Gbps / ~500 MB/s), which is still
  a significant improvement over the SD card.

**Validating the connection:**

After plugging the hub and SSD into one of the Pi 5's USB 3.0 ports (the blue
ports), verify the SSD is negotiating at USB 3.0 (5000 Mbps) rather than
falling back to USB 2.0 (480 Mbps):

```bash
lsusb -t
```

Look for the Crucial device in the tree and confirm the speed reads **5000M**.
If it shows **480M** the device has fallen back to USB 2.0 — try a different
cable, port, or check that the hub is connected to a blue USB 3.0 port on the
Pi.

**Planned approach:**
- Attach SSD via the Plugable USB 3.0 hub to a Pi 5 USB 3.0 port
- Create an Ansible role that formats and mounts the drive at a deterministic
  path (e.g. `/mnt/ssd`)
- Update Immich's `docker-compose.yml` volume mounts to point to the SSD
- Update the OMV shared-folder configuration via the OMV API

---

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

## Contributing

Have an idea not on this list? Open an issue or pull request!
