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
| Plugable 7-Port USB 3.0 Hub w/ 36 W power adapter | Chosen for reliable power delivery to the SSD; the 36 W adapter ensures the hub can power bus-hungry USB 3.0 devices without back-powering the Pi |
| Crucial X9 1 TB Portable SSD (USB 3.2 Gen 2, USB-C) | External storage for Immich media and database; connects to the Pi through the Plugable hub. Rated up to 1050 MB/s, limited to ~5 Gbps (USB 3.0) through the hub and Pi 5 USB 3.0 ports |
| Ethernet cable | **Required** for initial setup (WiFi can be configured later) |
| Control machine with VS Code and Docker | Uses devcontainer (no local Python/Ansible needed) |

---

## Step 1 — Flash Raspberry Pi OS Lite

1. Download **Raspberry Pi Imager** from <https://www.raspberrypi.com/software/>.
2. Choose **Raspberry Pi OS Lite (64-bit)** (Bookworm).
3. Before writing, open **Advanced options** (`Ctrl+Shift+X`) and:
   - Set a hostname, e.g. `homeserver`
   - Enable SSH (use password or public-key auth)
   - Set username and password
   - **Do NOT configure Wi-Fi** — use ethernet for initial setup
4. Flash to the SD card, insert into the Pi.
5. **Connect the Pi to your router via ethernet cable** before powering on.
6. Power on the Pi and wait ~60 seconds, then confirm SSH access:

   ```bash
   ssh pi@homeserver.local
   # or use the IP address shown in your router's DHCP table
   ```

7. **Recommended:** assign a static IP (or DHCP reservation) for the Pi in your
   router before continuing.

> **Why ethernet?** The OpenMediaVault installation reconfigures network management
> on the Pi. Using ethernet ensures uninterrupted connectivity during setup. WiFi
> can be configured afterward through the OMV web UI under **Network → Interfaces**.

---

## Step 2 — Prepare the control machine

This project uses a **devcontainer** so you don't need to install Python or Ansible locally.

### Prerequisites

- [Visual Studio Code](https://code.visualstudio.com/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine on Linux)
- [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) for VS Code

### Setup

1. Clone this repository:

   ```bash
   git clone https://github.com/tmarquard1/home-stash.git
   cd home-stash
   ```

2. Open the project in VS Code:

   ```bash
   code .
   ```

3. When prompted, click **"Reopen in Container"** (or press `Cmd+Shift+P` / `Ctrl+Shift+P` and select **"Dev Containers: Reopen in Container"**)

4. VS Code will build the container and install Ansible automatically. This may take a few minutes the first time.

5. Your SSH keys from `~/.ssh` are automatically mounted read-only into the container for Ansible to use.

> **Note:** If you prefer to install Ansible locally instead, you can do so with `brew install ansible` (macOS) or `apt install ansible` (Debian/Ubuntu), then install collections with `ansible-galaxy collection install community.general ansible.posix`.

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
  vars:
    ansible_python_interpreter: /usr/bin/python3
    immich_db_password: "your-strong-password-here"   # REQUIRED
```

> **Important:** You must set `immich_db_password` to a strong, unique password.
> The playbook will refuse to run if it is left as the default placeholder.

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
3. **Install Docker** — installs Docker Engine and CLI from Debian packages,
   then installs the `openmediavault-compose` UI plugin.
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
| `immich_db_password` fail | Set `immich_db_password` in your inventory (see Step 3) |
| OMV web UI not loading after playbook | Wait 2–3 min for `openmediavault` service to fully start |
| Immich containers exit immediately | Check `docker compose logs` on the Pi; often a missing env var |
| Docker not found after OMV-Extras install | Reboot the Pi and re-run `playbooks/immich.yml` |
| Lost connectivity during OMV install | OMV reconfigures networking - use ethernet; if using WiFi, manually restore with `sudo dhclient wlan0` |
