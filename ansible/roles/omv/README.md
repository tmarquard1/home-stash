# OMV Role

This Ansible role installs and configures **OpenMediaVault (OMV)** on Raspberry Pi OS, including OMV-Extras, Docker support, and external storage management.

---

## Purpose

OpenMediaVault is a network-attached storage (NAS) solution that provides:
- Web-based storage management
- Filesystem mounting and configuration
- Shared folder management
- Plugin ecosystem (Docker, Portainer, etc.)
- S.M.A.R.T. monitoring and disk health tracking

This role automates the complete setup of OMV on a fresh Raspberry Pi, including:
1. System preparation (package updates, prerequisite installation)
2. OMV installation via the official install script
3. OMV-Extras installation for Docker/Compose plugin support
4. External storage configuration (optional)

---

## Requirements

- **OS:** Raspberry Pi OS Lite (64-bit), Debian Bookworm or later
- **Hardware:** Raspberry Pi 4 or 5 (tested on Pi 5 with 4GB/8GB RAM)
- **Network:** SSH access with sudo privileges
- **Ansible:** 2.14 or later
- **Collections:**
  - `community.general` (for parted and filesystem modules)
  - `ansible.posix`

Install required collections:

```bash
ansible-galaxy collection install community.general ansible.posix
```

---

## Role Variables

### OMV Installation

Defined in `defaults/main.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `omv_install_script_url` | [GitHub URL] | URL of the official OMV install script |
| `omv_extras_install_script_url` | [GitHub URL] | URL of the OMV-Extras install script |
| `omv_service_wait_seconds` | `120` | Seconds to wait for OMV services to stabilize after install |
| `omv_install_compose_plugin` | `true` | Whether to install Docker Compose plugin via OMV-Extras |

### External Storage

| Variable | Default | Description |
|----------|---------|-------------|
| `omv_setup_external_storage` | `true` | Enable automatic external storage setup |
| `omv_external_disk_device` | `/dev/sda` | Device path of the external disk (verify with `lsblk`) |
| `omv_external_disk_label` | `immich-storage` | Filesystem label for identification |
| `omv_immich_shared_folder_name` | `immich` | Shared folder name for Immich data |

---

## Dependencies

None. This role is self-contained.

---

## Example Playbook

```yaml
---
- name: Set up OpenMediaVault on Raspberry Pi
  hosts: homeserver
  become: true
  roles:
    - role: omv
      vars:
        omv_setup_external_storage: true
        omv_external_disk_device: /dev/sda
```

---

## Tasks

The role is organized into modular task files:

### 1. `prepare.yml` — System Preparation

- Updates the package cache
- Installs prerequisites (curl, wget, gnupg)
- Ensures the system is ready for OMV installation

### 2. `install_omv.yml` — OMV Installation

- Checks if OMV is already installed (idempotent)
- Downloads and runs the official OMV install script
- Waits for OMV services to stabilize
- Ensures `openmediavault-engined` is running

### 3. `install_extras.yml` — OMV-Extras and Docker

- Installs OMV-Extras plugin repository
- Installs Docker Engine via OMV-Extras
- Optionally installs the `openmediavault-compose` plugin for Docker Compose support

### 4. `setup_storage.yml` — External Storage Configuration

- Detects the external disk specified in `omv_external_disk_device`
- Creates a GPT partition table (if disk is blank)
- Formats the partition with ext4 and applies the specified label
- Registers the filesystem with OMV using `omv-salt deploy run fstab`
- Creates Immich subdirectories (`library/`, `postgres/`)
- Sets appropriate ownership for Docker container access
- Exports Ansible facts (`omv_immich_library_path`, `omv_immich_db_path`) for use by other roles

---

## Storage Management

### How It Works

When `omv_setup_external_storage: true`:

1. The role detects the external disk (e.g., `/dev/sda`)
2. If the disk is unpartitioned, it creates a single GPT partition
3. If the partition has no filesystem, it formats with ext4
4. OMV's `omv-salt deploy run fstab` command detects and mounts the filesystem at:
   ```
   /srv/dev-disk-by-uuid-{UUID}/
   ```
5. Subdirectories are created for Immich:
   ```
   /srv/dev-disk-by-uuid-{UUID}/immich/
   ├── library/
   └── postgres/
   ```
6. The paths are stored as Ansible facts for the Immich role to consume

### Idempotency

- **Partitioning** only occurs if the disk has no partition table
- **Formatting** only occurs if the partition has no filesystem
- Existing filesystems and data are **never overwritten**
- Running the role multiple times is safe

### Disabling External Storage

Set `omv_setup_external_storage: false` to skip storage configuration. Immich will then use SD card storage at `/opt/immich/`.

---

## Facts Exported

This role sets the following facts when external storage is configured:

| Fact | Description | Example |
|------|-------------|---------|
| `omv_immich_storage_path` | Base path for Immich data | `/srv/dev-disk-by-uuid-abc123/immich` |
| `omv_immich_library_path` | Path for Immich photo/video library | `/srv/dev-disk-by-uuid-abc123/immich/library` |
| `omv_immich_db_path` | Path for PostgreSQL database | `/srv/dev-disk-by-uuid-abc123/immich/postgres` |

These facts are consumed by the `immich` role to configure Docker Compose volume mounts.

---

## Handlers

Defined in `handlers/main.yml`:

| Handler | Trigger | Action |
|---------|---------|--------|
| `restart omv-engined` | OMV configuration changes | Restarts the OMV engine daemon |

---

## Tags

Use tags for selective execution:

| Tag | Tasks |
|-----|-------|
| `storage` | External storage setup only |
| `immich` | Storage tasks relevant to Immich |

Example:

```bash
ansible-playbook playbooks/site.yml --tags=storage
```

---

## OMV Web UI

After installation, access the OMV web interface:

- **URL:** `http://{pi-hostname}` or `http://{pi-ip-address}`
- **Default username:** `admin`
- **Default password:** `openmediavault`

**⚠️ Security:** Change the default password immediately after first login.

---

## Troubleshooting

### OMV Installation Fails

**Symptom:** Install script exits with an error

**Solutions:**
1. Check Raspberry Pi OS version: `cat /etc/os-release`
   - OMV 7 requires Debian Bookworm (12) or later
2. Ensure sufficient disk space: `df -h`
3. Check logs: `journalctl -xe`

### External Disk Not Detected

**Symptom:** Playbook fails with "External disk /dev/sda not found"

**Solutions:**
1. Verify the disk is connected: `lsblk`
2. Check USB connection: `lsusb -t`
3. Update `omv_external_disk_device` if the device path differs

### OMV Web UI Not Accessible

**Symptom:** Cannot reach `http://{pi-ip}`

**Solutions:**
1. Verify OMV services are running:
   ```bash
   systemctl status openmediavault-engined
   systemctl status nginx
   ```
2. Check firewall rules: `sudo iptables -L`
3. Access via IP address instead of hostname

---

## Security Considerations

- **Change default OMV password** on first login
- **Firewall:** OMV web UI is exposed on port 80/443 by default—consider restricting access
- **External storage:** The ext4 filesystem is unencrypted—use LUKS for encryption if needed
- **Updates:** Keep OMV updated via the web UI: **System > Update Management**

---

## References

- [OpenMediaVault Documentation](https://docs.openmediavault.org/)
- [OMV Install Script](https://github.com/OpenMediaVault-Plugin-Developers/installScript)
- [OMV-Extras Plugin](https://github.com/OpenMediaVault-Plugin-Developers/packages)
- [Storage Configuration Guide](../docs/storage.md)

---

## License

See [LICENSE](../../LICENSE) in the project root.
