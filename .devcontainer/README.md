# Devcontainer Configuration

This devcontainer provides a pre-configured development environment for running Ansible playbooks without installing Python or Ansible on your local machine.

## What's included

- **Python 3.12** - Base runtime for Ansible
- **Ansible** - Installed via pip during container creation
- **Ansible Collections** - `community.general` and `ansible.posix` pre-installed
- **VS Code Extensions**:
  - `redhat.ansible` - Ansible language support
  - `redhat.vscode-yaml` - YAML syntax highlighting
- **SSH Key Mounting** - Your `~/.ssh` directory is mounted read-only for authentication with target hosts

## Usage

1. Install [VS Code](https://code.visualstudio.com/) and [Docker Desktop](https://www.docker.com/products/docker-desktop/)
2. Install the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
3. Open this project in VS Code
4. Click **"Reopen in Container"** when prompted (or use the command palette: `Dev Containers: Reopen in Container`)
5. Wait for the container to build and Ansible to install (~2-3 minutes first time)
6. Run Ansible commands from the integrated terminal as usual

## Customization

To modify the devcontainer configuration, edit [`.devcontainer/devcontainer.json`](devcontainer.json).

Common customizations:
- Add more VS Code extensions to `customizations.vscode.extensions`
- Install additional Python packages in `postCreateCommand`
- Add additional Ansible collections to the `postCreateCommand`
