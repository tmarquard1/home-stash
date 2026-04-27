# External Storage Configuration

This document explains how external storage (SSD/HDD) is configured and managed for Immich and other services using OpenMediaVault.

---

## Overview

The home-stash project uses **OpenMediaVault (OMV)** to manage external storage devices. OMV provides:
- Automatic filesystem detection and mounting
- Web UI for storage management
- Persistent mount configuration via `/etc/fstab`
- Shared folder management for applications

When external storage is enabled, Immich's photo library and PostgreSQL database are stored on the external drive instead of the SD card, improving performance and longevity.

---

## Hardware Setup

### Validated Configuration

- **SSD:** Crucial X9 1TB Portable SSD (USB 3.2 Gen 2)
- **USB Hub:** Plugable 7-Port USB 3.0 Hub with 36W power adapter
- **Connection:** Hub connected to Raspberry Pi 5 USB 3.0 port (blue port)

### Verifying USB 3.0 Connection

After connecting the SSD, verify it's negotiating at USB 3.0 speeds:

```bash
lsusb -t | grep -A2 "5000M"
```

You should see your storage device listed with **5000M** speed. If it shows **480M**, the device has fallen back to USB 2.0—check cables, ports, and hub power.

### Identifying the Drive

To find the device path of your SSD:

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

Look for your SSD model (e.g., `CT1000X9SSD9` for Crucial X9). The device will typically be `/dev/sda`, `/dev/sdb`, etc.

---

## Ansible Configuration

### Enabling External Storage

External storage is configured in the OMV role. Edit the variables in your inventory or `ansible/roles/omv/defaults/main.yml`:

```yaml
# External storage configuration
omv_setup_external_storage: true        # Set to false to disable

# Device path of the external disk (verify with lsblk)
omv_external_disk_device: /dev/sda

# Filesystem label for identification
omv_external_disk_label: immich-storage

# Shared folder name for Immich data
omv_immich_shared_folder_name: immich
```

### What the Playbook Does

When you run the playbook with external storage enabled:

1. **Partitions the disk** (if not already partitioned) using GPT
2. **Creates an ext4 filesystem** with the specified label
3. **Registers the filesystem with OMV** using `omv-salt deploy run fstab`
4. **Mounts the drive** at `/srv/dev-disk-by-uuid-{UUID}/`
5. **Creates subdirectories** for Immich:
   - `library/` — photo and video uploads
   - `postgres/` — PostgreSQL database
6. **Sets ownership** to the Ansible user for Docker container access
7. **Passes paths to Immich role** via Ansible facts
8. **Migrates existing data** from `/opt/immich/` to SSD (if `immich_migrate_existing_data: true`)
   - Automatically detects existing data on SD card
   - Stops Immich containers during migration
   - Copies data with integrity verification
   - Only runs if new location is empty (fully idempotent)
9. **Restarts Immich** with new storage configuration

### Running the Playbook

To set up or verify external storage:

```bash
cd ansible
ansible-playbook playbooks/site.yml
```

To run only storage-related tasks:

```bash
ansible-playbook playbooks/site.yml --tags=storage
```

---

## Storage Paths

Once configured, storage paths follow OMV's standard pattern:

```
/srv/dev-disk-by-uuid-{UUID}/immich/
├── library/          # Immich photo/video uploads
└── postgres/         # PostgreSQL database
```

The exact UUID is determined at runtime by OMV when the filesystem is registered.

### Viewing in OMV Web UI

Access the OpenMediaVault web interface at `http://{pi-hostname}` or `http://{pi-ip}`:

1. **Storage > File Systems** — Shows mounted filesystems and UUIDs
2. **Storage > Shared Folders** — Shows the `immich` shared folder configuration

---

## Idempotency and Re-running

The storage setup tasks are **idempotent**:
- Running the playbook multiple times is safe
- Partitioning and formatting only occur if the disk is blank
- Existing filesystems and data are **never** overwritten
- OMV configuration is updated only when necessary

---

## Automatic Data Migration

The playbook includes **automatic migration** of existing Immich data from the SD card to external storage.

### How It Works

When `immich_migrate_existing_data: true` (default):

1. **Detection:** Checks if data exists in `/opt/immich/library` or `/opt/immich/postgres`
2. **Verification:** Confirms the new SSD location is empty (won't overwrite existing data)
3. **Migration:** If needed:
   - Stops Immich containers
   - Copies data using `rsync` with checksums
   - Verifies data integrity (compares sizes)
   - Fixes ownership for Docker access
   - Containers restart with new storage

### Migration is Fully Idempotent

- **First run:** Migrates data if found on SD card
- **Subsequent runs:** Skips migration (data already on SSD)
- **Safe to re-run:** Won't duplicate or overwrite data

### Migration Time Estimates

| Data Size | Estimated Time |
|-----------|----------------|
| < 10 GB | 5-10 minutes |
| 10-50 GB | 10-30 minutes |
| 50-100 GB | 30-60 minutes |
| > 100 GB | 1+ hours |

### Disabling Automatic Migration

To skip migration and start fresh:

```yaml
# In your inventory or playbook
immich_migrate_existing_data: false
```

### Old Data Cleanup

After successful migration, the original data remains in `/opt/immich/` as a backup:

```bash
# After confirming everything works (wait 1-2 weeks)
ssh user@pi
sudo rm -rf /opt/immich/library /opt/immich/postgres
```

See [docs/roadmap.md](roadmap.md) item #9 for automated cleanup plans.

---

## Troubleshooting

### Drive Not Detected

**Symptom:** Playbook fails with "External disk /dev/sda not found"

**Solutions:**
1. Verify the drive is connected: `lsblk`
2. Check USB connection: `lsusb -t`
3. Update `omv_external_disk_device` in inventory if the device path differs

### Drive Not Mounting

**Symptom:** Filesystem created but not mounted

**Solutions:**
1. Check OMV logs: `journalctl -u openmediavault-engined -f`
2. Manually deploy fstab: `sudo omv-salt deploy run fstab`
3. Verify mount point exists: `ls -la /srv/`

### Immich Not Using SSD

**Symptom:** Immich still writes to SD card (`/opt/immich`)

**Solutions:**
1. Verify storage facts were set:
   ```bash
   ansible -m debug -a "var=omv_immich_library_path" homeserver
   ```
2. Check Immich `.env` file:
   ```bash
   ssh user@pi 'cat /opt/immich/.env | grep LOCATION'
   ```
3. Redeploy Immich:
   ```bash
   ansible-playbook playbooks/immich.yml
   ```

### Permission Errors in Docker

**Symptom:** Immich containers can't write to SSD

**Solutions:**
1. Verify directory ownership:
   ```bash
   ssh user@pi 'ls -la /srv/dev-disk-by-uuid-*/immich/'
   ```
2. Ownership should match the `ansible_user` (typically `pi` or your SSH user)
3. If incorrect, re-run with `--tags=storage`

---

## Disabling External Storage

To revert to SD card storage:

1. Set `omv_setup_external_storage: false` in your inventory
2. Re-run the playbook: `ansible-playbook playbooks/site.yml`
3. Immich will use `/opt/immich/library` and `/opt/immich/postgres` again

**Note:** Existing data on the SSD is not deleted. You'll need to manually migrate data if needed.

---

## Adding More Services

To use the SSD for additional services (e.g., Nextcloud, Plex):

1. Create additional subdirectories in the OMV storage setup tasks:
   ```yaml
   - name: Create additional service directories
     ansible.builtin.file:
       path: "/srv/dev-disk-by-uuid-{{ disk_uuid.stdout }}/{{ item }}"
       state: directory
       owner: "{{ ansible_user }}"
       group: "{{ ansible_user }}"
       mode: "0755"
     loop:
       - nextcloud
       - plex
   ```

2. Pass paths to service roles via facts, similar to Immich

3. Update service Docker Compose files to mount these paths

---

## Security Considerations

- **No encryption:** The ext4 filesystem is unencrypted. For encrypted storage, consider LUKS before formatting.
- **Physical security:** Anyone with physical access can remove and read the drive.
- **Backups:** External storage is not backed up by default. See [docs/roadmap.md](roadmap.md) for encrypted cloud backup plans.
- **User access:** Directories are owned by the Ansible user. Docker containers run with this user's UID/GID.

---

## Performance Notes

### Expected Speeds

- **SD Card (typical):** 20-40 MB/s random I/O
- **USB 3.0 SSD:** 200-400 MB/s random I/O (depending on SSD and USB hub)
- **Crucial X9 via USB 3.0:** ~300-400 MB/s sequential, ~150-250 MB/s random

### Monitoring Performance

Benchmark read/write speeds:

```bash
# Write test (creates 1GB file)
sudo dd if=/dev/zero of=/srv/dev-disk-by-uuid-*/test.img bs=1M count=1024 conv=fdatasync

# Read test
sudo dd if=/srv/dev-disk-by-uuid-*/test.img of=/dev/null bs=1M count=1024

# Clean up
sudo rm /srv/dev-disk-by-uuid-*/test.img
```

Check disk I/O in real-time:

```bash
sudo iotop
```

---

## Future Enhancements

See [docs/roadmap.md](roadmap.md) for planned storage-related features:
- Boot from SSD (eliminate SD card entirely)
- Encrypted cloud backups
- Multi-disk RAID configurations
- Automatic disk health monitoring (SMART)

---

## References

- [OpenMediaVault Documentation](https://docs.openmediavault.org/)
- [Ansible parted module](https://docs.ansible.com/ansible/latest/collections/community/general/parted_module.html)
- [Ansible filesystem module](https://docs.ansible.com/ansible/latest/collections/community/general/filesystem_module.html)
- [Raspberry Pi Storage Documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-5-2)
