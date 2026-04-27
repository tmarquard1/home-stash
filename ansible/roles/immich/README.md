# Immich Role

This Ansible role deploys **Immich** (self-hosted photo and video management) on Raspberry Pi using Docker Compose.

---

## Purpose

Immich is a high-performance self-hosted alternative to Google Photos, providing:
- Photo and video upload/backup from mobile devices
- Smart search and AI-powered face recognition
- Timeline-based organization
- Sharing and collaboration features
- Machine learning for object detection and classification

This role automates the complete deployment including:
1. Directory structure creation
2. Docker Compose configuration
3. Automatic data migration from SD card to external storage
4. Container orchestration

---

## Requirements

- **OS:** Raspberry Pi OS Lite (64-bit), Debian Bookworm or later
- **Hardware:** Raspberry Pi 4 or 5 (minimum 4GB RAM recommended)
- **Docker:** Docker Engine and Docker Compose installed (provided by `omv` role)
- **Storage:** External SSD recommended for photo library and database (optional)
- **Ansible:** 2.14 or later
- **Collections:**
  - `community.docker` (for docker_compose_v2 module)
  - `ansible.posix` (for synchronize module)

Install required collections:

```bash
ansible-galaxy collection install community.docker ansible.posix
```

---

## Role Variables

### Basic Configuration

Defined in `defaults/main.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `immich_base_dir` | `/opt/immich` | Base directory for Immich configuration |
| `immich_version` | `v2.7.5` | Immich version to deploy (see [releases](https://github.com/immich-app/immich/releases)) |
| `immich_port` | `2283` | Web interface port |
| `immich_timezone` | `America/New_York` | Timezone for containers (TZ database name) |
| `immich_enable_ml` | `true` | Enable machine learning container (set `false` to save RAM) |

### Database Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `immich_db_name` | `immich` | PostgreSQL database name |
| `immich_db_username` | `immich` | PostgreSQL username |
| `immich_db_password` | `CHANGEME_SET_IN_VAULT` | **Must be overridden** with Ansible Vault |

**⚠️ Security:** You **must** override `immich_db_password` via Ansible Vault or `--extra-vars`. The playbook will fail if the default placeholder is detected.

### Storage Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `immich_upload_location` | Auto-detected | Photo/video library path (uses SSD if available) |
| `immich_db_data_location` | Auto-detected | PostgreSQL data path (uses SSD if available) |

**Note:** When external storage is configured via the `omv` role, these paths are automatically set to the SSD location. Otherwise, they default to subdirectories under `immich_base_dir` on the SD card.

### Data Migration

| Variable | Default | Description |
|----------|---------|-------------|
| `immich_migrate_existing_data` | `true` | Automatically migrate data from SD card to external storage |

When enabled, the role will:
- Detect existing data in `/opt/immich/library` and `/opt/immich/postgres`
- Copy it to the new storage location (if empty)
- Verify data integrity
- Keep the original data as backup

---

## Dependencies

This role works best with the `omv` role for storage management, but can be used standalone.

When using external storage:
1. Run the `omv` role first to set up the SSD
2. The `omv` role exports `omv_immich_library_path` and `omv_immich_db_path` facts
3. This role automatically uses those paths

---

## Example Playbook

### Basic Deployment (SD Card Storage)

```yaml
---
- name: Deploy Immich
  hosts: homeserver
  become: true
  roles:
    - role: immich
      vars:
        immich_db_password: "{{ vault_immich_db_password }}"
```

### With External Storage (Recommended)

```yaml
---
- name: Deploy Immich with SSD storage
  hosts: homeserver
  become: true
  roles:
    - role: omv
      vars:
        omv_setup_external_storage: true
        omv_external_disk_device: /dev/sda
    - role: immich
      vars:
        immich_db_password: "{{ vault_immich_db_password }}"
        immich_migrate_existing_data: true
```

---

## Tasks Overview

### Main Tasks (`tasks/main.yml`)

1. **Validate configuration** — Ensures `immich_db_password` is set
2. **Create directories** — Base dir, library, database locations
3. **Template configuration** — `.env` file and `docker-compose.yml`
4. **Migrate data** — (if enabled) Copies existing data to new storage
5. **Deploy containers** — Pulls images and starts services

### Migration Tasks (`tasks/migrate_data.yml`)

1. **Detect old data** — Checks `/opt/immich/` for existing library/database
2. **Check new location** — Ensures SSD destination is empty (idempotent)
3. **Stop containers** — Safely stops Immich before migration
4. **Copy data** — Uses `rsync` with checksums for integrity
5. **Verify migration** — Compares file sizes (95%+ match required)
6. **Fix ownership** — Sets proper permissions for Docker containers
7. **Restore service** — Containers restart with new storage

---

## Tags

Use tags for selective execution:

| Tag | Tasks |
|-----|-------|
| `migration` | Data migration tasks only |
| `storage` | Storage-related tasks (migration, directory setup) |

Example:

```bash
# Run only migration tasks
ansible-playbook playbooks/immich.yml --tags=migration

# Skip migration
ansible-playbook playbooks/immich.yml --skip-tags=migration
```

---

## Handlers

Defined in `handlers/main.yml`:

| Handler | Trigger | Action |
|---------|---------|--------|
| `Restart Immich` | Configuration file changes | Restarts all Immich containers via Docker Compose |

---

## Accessing Immich

After deployment:

- **URL:** `http://{pi-hostname}:2283` or `http://{pi-ip}:2283`
- **First visit:** Create an admin account (first user becomes admin)
- **Mobile apps:** Available for iOS and Android
  - [iOS App Store](https://apps.apple.com/app/immich/id1613945652)
  - [Google Play Store](https://play.google.com/store/apps/details?id=app.alextran.immich)

---

## Storage Considerations

### SD Card vs. SSD

| Aspect | SD Card | External SSD |
|--------|---------|--------------|
| **Speed** | 20-40 MB/s | 200-400 MB/s |
| **Longevity** | Limited write cycles | Much higher endurance |
| **Recommendation** | Testing only | Production use |

**Bottom line:** Use an external SSD for any meaningful photo library. The `omv` role handles SSD setup automatically.

### Data Migration

The automatic migration feature is:
- **Idempotent:** Safe to run multiple times
- **Verified:** Checks data integrity after copying
- **Non-destructive:** Original data remains as backup
- **Incremental:** Uses `rsync` for efficient copying

Migration time depends on data size (see [docs/storage.md](../../docs/storage.md)).

---

## Machine Learning

Immich includes a machine learning container for:
- Face detection and recognition
- Object classification
- Smart search

**Resource usage:**
- Adds ~500MB-1GB RAM usage
- Uses CPU for inference (Pi 5 recommended)
- Can be disabled: `immich_enable_ml: false`

For advanced AI features, see [docs/roadmap.md](../../docs/roadmap.md) item #7.

---

## Troubleshooting

### Database Password Error

**Symptom:** Playbook fails with "immich_db_password is still set to the default placeholder"

**Solution:** Override the password in your inventory or via Ansible Vault:

```yaml
# inventory/hosts.yml
all:
  vars:
    immich_db_password: "{{ vault_immich_db_password }}"
```

Create a vaulted variable:

```bash
ansible-vault create group_vars/all/vault.yml
# Add: vault_immich_db_password: "your-secure-password"
```

### Migration Fails with Size Mismatch

**Symptom:** "Library migration verification failed"

**Solutions:**
1. Check disk space on SSD: `df -h`
2. Verify rsync completed: Check for error messages
3. Re-run the playbook (migration is idempotent)

### Containers Won't Start

**Symptom:** `docker compose ps` shows containers as unhealthy

**Solutions:**
1. Check logs: `docker compose -f /opt/immich/docker-compose.yml logs`
2. Verify database credentials in `.env` file
3. Ensure storage paths are accessible and writable

### Permission Denied Errors

**Symptom:** Immich can't read/write to storage

**Solutions:**
1. Check ownership: `ls -la /srv/dev-disk-by-uuid-*/immich/`
2. Re-run playbook to fix permissions
3. Verify Docker containers run as correct user

---

## Security Considerations

- **Database password:** Always use Ansible Vault for `immich_db_password`
- **Network exposure:** By default, Immich listens on all interfaces (0.0.0.0)
  - Use a reverse proxy or firewall to restrict access
  - See [docs/roadmap.md](../../docs/roadmap.md) item #5 for Tailscale VPN
- **Updates:** Keep Immich updated by changing `immich_version` and re-running the playbook
- **Backups:** See [docs/roadmap.md](../../docs/roadmap.md) items #3 and #10 for backup strategies

---

## Version Management

To upgrade Immich:

1. Check [release notes](https://github.com/immich-app/immich/releases)
2. Update `immich_version` in your inventory:
   ```yaml
   immich_version: "v2.8.0"
   ```
3. Re-run the playbook:
   ```bash
   ansible-playbook playbooks/immich.yml
   ```

**Note:** Always read release notes for breaking changes and migration steps.

---

## References

- [Immich Documentation](https://immich.app/docs)
- [Immich GitHub Repository](https://github.com/immich-app/immich)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Storage Configuration Guide](../../docs/storage.md)
- [Project Roadmap](../../docs/roadmap.md)

---

## License

See [LICENSE](../../LICENSE) in the project root.
