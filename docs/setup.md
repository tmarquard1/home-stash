# Setup Guide

This guide walks through every step needed to go from a bare Raspberry Pi 5
running Raspberry Pi OS Lite to a fully operational NAS with Immich running
inside Docker containers managed by OpenMediaVault.

---

## Prerequisites

| Item | Notes |
|------|-------|
| Raspberry Pi 5 (4 GB or 8 GB RAM recommended) | |
| MicroSD card ≥ 16 GB (Class 10 / A2 rated) | Temporary boot medium |
| Power supply (official 27 W USB-C) | |
| Ethernet cable | Recommended for reliability |
| Control machine (macOS/Linux/WSL) | Needs Ansible ≥ 2.14 and Python ≥ 3.10 |

---

## Step 1 — Flash Raspberry Pi OS Lite

1. Download **Raspberry Pi Imager** from <https://www.raspberrypi.com/software/>.
2. Choose **Raspberry Pi OS Lite (64-bit)** (Bookworm).
3. Before writing, open **Advanced options** (`Ctrl+Shift+X`) and:
   - Set a hostname, e.g. `homeserver`
   - Enable SSH (use password or public-key auth)
   - Set username and password
   - Configure Wi-Fi only if you cannot use Ethernet
4. Flash to the SD card, insert into the Pi and power on.
5. Wait ~60 seconds, then confirm SSH access:

   ```bash
   ssh pi@homeserver.local
   # or use the IP address shown in your router's DHCP table
   ```

6. **Recommended:** assign a static IP (or DHCP reservation) for the Pi in your
   router before continuing.

---

## Step 2 — Prepare the control machine

Install Ansible and the required collections on your laptop/desktop:

```bash
# macOS (Homebrew)
brew install ansible

# Debian/Ubuntu
sudo apt update && sudo apt install -y ansible

# Install required Ansible collections
ansible-galaxy collection install community.general ansible.posix
```

Clone this repository (if you haven't already):

```bash
git clone https://github.com/tmarquard1/home-stash.git
cd home-stash
```

---

## Step 3 — Configure the inventory

Copy the example inventory and set the Pi's address:

```bash
cp ansible/inventory/hosts.yml.example ansible/inventory/hosts.yml
```

Edit `ansible/inventory/hosts.yml`:

```yaml
all:
  hosts:
    homeserver:
      ansible_host: 192.168.1.100   # <-- your Pi's IP
      ansible_user: pi              # <-- the user you set in Imager
```

Test connectivity:

```bash
cd ansible
ansible -i inventory/hosts.yml all -m ping
```

---

## Step 4 — Run the full playbook

```bash
cd ansible
ansible-playbook playbooks/site.yml
```

The playbook will:

1. **System preparation** — update packages, set hostname, configure a firewall
   baseline, enable `cgroups` for container support.
2. **Install OpenMediaVault** — runs the official OMV install script, waits for
   the service to stabilise, then installs OMV-Extras.
3. **Install Docker via OMV-Extras** — installs the `openmediavault-compose`
   plugin which provides Docker Engine and Docker Compose.
4. **Deploy Immich** — copies the Compose stack and `.env` file to the Pi,
   pulls images, and starts the services.

> **Total runtime:** 15–30 minutes depending on your internet connection.

---

## Step 5 — Post-install checks

### OpenMediaVault

Open `http://<pi-ip>` in a browser.

Default credentials:

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `openmediavault` |

**Change the password immediately** via **System → General Settings → Web
Administrator Password**.

### Immich

Open `http://<pi-ip>:2283` in a browser.

On first launch, complete the **Immich onboarding wizard** to create your
admin account.

---

## Step 6 — Optional: harden SSH

After confirming Ansible can log in with your SSH key, disable password
authentication:

```bash
# On the Pi
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' \
  /etc/ssh/sshd_config
sudo systemctl restart sshd
```

---

## Directory layout

```
ansible/
├── ansible.cfg                  # Ansible defaults
├── inventory/
│   ├── hosts.yml                # Your inventory (git-ignored)
│   └── hosts.yml.example        # Template
├── playbooks/
│   ├── site.yml                 # Master playbook (runs everything)
│   ├── omv.yml                  # OMV + Docker install only
│   └── immich.yml               # Immich deploy only
└── roles/
    ├── omv/                     # OpenMediaVault role
    └── immich/                  # Immich role
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `UNREACHABLE` on first `ansible ping` | Confirm SSH works manually; check IP in `hosts.yml` |
| OMV web UI not loading after playbook | Wait 2–3 min for `openmediavault` service to fully start |
| Immich containers exit immediately | Check `docker compose logs` on the Pi; often a missing env var |
| Docker not found after OMV-Extras install | Reboot the Pi and re-run `playbooks/immich.yml` |
