# home-stash

A fully automated setup for a **Raspberry Pi 5** home server running
[OpenMediaVault](https://www.openmediavault.org/) with Docker extras and
[Immich](https://immich.app/) for self-hosted photo and video management.

## What's included

| Layer | Technology |
|-------|-----------|
| OS | Raspberry Pi OS Lite (64-bit) |
| NAS/Media OS | OpenMediaVault 7 (OMV) |
| Container runtime | Docker (via OMV-Extras) |
| Photo management | Immich |

## Quick start

1. Flash **Raspberry Pi OS Lite (64-bit)** to an SD card and boot the Pi.
2. Enable SSH and note the Pi's IP address (set a static IP or reserve it in your router).
3. Copy and edit the inventory:

   ```bash
   cp ansible/inventory/hosts.yml.example ansible/inventory/hosts.yml
   # edit hosts.yml and set your Pi's IP / hostname
   ```

4. Run the full site playbook:

   ```bash
   cd ansible
   ansible-playbook playbooks/site.yml
   ```

5. When the playbook finishes:
   - OpenMediaVault web UI → `http://<pi-ip>` (default creds: `admin` / `openmediavault`)
   - Immich web UI → `http://<pi-ip>:2283`

See [`docs/setup.md`](docs/setup.md) for the detailed step-by-step guide and
[`docs/roadmap.md`](docs/roadmap.md) for planned future enhancements.
