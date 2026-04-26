# home-stash

A fully automated setup for a **Raspberry Pi 5** home server running
[OpenMediaVault](https://www.openmediavault.org/) with Docker extras and
[Immich](https://immich.app/) for self-hosted photo and video management.

## What's included

| Layer | Technology |
|-------|-----------|
| OS | Raspberry Pi OS Lite (64-bit) |
| NAS/Media OS | OpenMediaVault 7 (OMV) |
| Container runtime | Docker (via Debian packages) |
| Photo management | Immich |

## Security & Quality Automation

This project includes automated security and quality checks:

- **🔒 CodeQL Security Scanning** - Continuous vulnerability analysis on every push
- **📦 Dependabot** - Automated dependency updates for GitHub Actions, Docker, and Python packages
- **✅ Ansible Lint** - Automated code quality checks for Ansible playbooks
- **📚 Comprehensive Documentation** - Copilot instructions ensure secure, idempotent development practices

All automation runs via GitHub Actions and keeps your infrastructure code secure and up-to-date.

## Quick start

> **💡 No local Python/Ansible required!** This project includes a devcontainer. Open in VS Code with Docker installed and select **"Reopen in Container"** to get started immediately.

1. Flash **Raspberry Pi OS Lite (64-bit)** to an SD card and boot the Pi.
2. **Connect the Pi to your router via ethernet cable** (WiFi can be configured later through OMV web UI).
3. Enable SSH and note the Pi's IP address (set a static IP or reserve it in your router).
4. Copy and edit the inventory:

   ```bash
   cp ansible/inventory/hosts.yml.example ansible/inventory/hosts.yml
   # edit hosts.yml and set your Pi's IP / hostname
   ```

5. Run the full site playbook:

   ```bash
   cd ansible
   ansible-playbook playbooks/site.yml
   ```

6. When the playbook finishes:
   - OpenMediaVault web UI → `http://<pi-ip>` (default creds: `admin` / `openmediavault`)
   - Immich web UI → `http://<pi-ip>:2283`

See [`docs/setup.md`](docs/setup.md) for the detailed step-by-step guide and
[`docs/roadmap.md`](docs/roadmap.md) for planned future enhancements.
