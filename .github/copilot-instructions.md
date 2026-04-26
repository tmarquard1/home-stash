# Copilot Instructions for home-stash

## Project Overview

This is an Ansible-based infrastructure-as-code project for automating a Raspberry Pi 5 home server running OpenMediaVault (OMV) and Immich. The project emphasizes **security**, **idempotency**, **re-runnability**, and **comprehensive documentation**.

---

## Core Principles

### 🔒 Security First

- **Never hardcode secrets**: Use Ansible Vault for sensitive data (passwords, API keys, tokens)
- **Principle of least privilege**: Create service accounts with minimal required permissions
- **Secure by default**: Enable firewalls, disable unnecessary services, use strong authentication
- **Review dependencies**: Verify all external roles, collections, and Docker images before use
- **Regular updates**: Keep all components (OS, packages, Docker images) up to date
- **Network isolation**: Use Docker networks appropriately; don't expose services unnecessarily

**Documentation references:**
- [Ansible Vault best practices](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
- [Docker security best practices](https://docs.docker.com/engine/security/)
- [OpenMediaVault security documentation](https://docs.openmediavault.org/)

### 🔄 Idempotency and Re-runnability

All Ansible tasks **must be idempotent**:
- Running the same playbook multiple times should produce the same result
- Use appropriate modules (`copy`, `template`, `file`) with `state` parameters
- Check before changing: use `changed_when`, `creates`, `unless` conditions
- Avoid shell commands when Ansible modules exist
- Test by running playbooks 2-3 times and verifying no changes occur after the first run

**Documentation references:**
- [Ansible best practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [Understanding idempotency](https://docs.ansible.com/ansible/latest/reference_appendices/glossary.html#term-Idempotency)

### 📖 Comprehensive Documentation

- **Every role** must have a `README.md` explaining its purpose, variables, and dependencies
- **Complex tasks** should include inline comments explaining the "why"
- **Document assumptions**: What OS? What must be pre-configured? What external dependencies?
- **Update project docs** (`docs/`) when adding features or changing workflows
- **Maintain roadmap**: Update `docs/roadmap.md` when completing or adding tasks
- **Include examples**: Provide working examples in inventory files and variable definitions

---

## When Suggesting Changes

### ❓ Always Ask Clarifying Questions

Before implementing any change, **challenge assumptions** and ask:

1. **Scope**: "Should this change apply to all hosts or just specific ones?"
2. **Security**: "Does this introduce any security risks? Should we use Ansible Vault?"
3. **Idempotency**: "How will this behave if run multiple times?"
4. **Dependencies**: "Are there prerequisites or ordering requirements?"
5. **Rollback**: "How can this be reverted if something goes wrong?"
6. **Testing**: "How can we validate this works without affecting production?"
7. **Documentation**: "What documentation needs to be updated?"
8. **Compatibility**: "Is this compatible with the current OS/versions in use?"

### 🏭 Industry Best Practices

Follow these standards:

**Ansible Structure:**
- Follow the [standard directory layout](https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html)
- Use roles for reusable components
- Keep playbooks simple and focused
- Use `ansible-lint` for code quality checks
- Tag tasks appropriately for selective execution
- Use handlers for service restarts/reloads

**Variable Naming:**
- Prefix role variables with role name (e.g., `immich_version`, `omv_admin_password`)
- Use snake_case for variable names
- Document all variables in `defaults/main.yml`
- Never use bare variables in templates (use `{{ variable_name }}`)

**Task Writing:**
- One task does one thing
- Use descriptive task names that explain the outcome, not the action
- Include `tags:` for logical grouping
- Use `block:` for error handling and grouping
- Specify `become:` explicitly when needed

**Docker/Compose:**
- Pin image versions (never use `:latest` in production)
- Use Docker Compose for multi-container applications
- Mount volumes explicitly; document data persistence locations
- Use environment files (`.env`) for configuration
- Implement health checks where appropriate

**Git Workflow:**
- Write clear, descriptive commit messages
- Keep commits atomic and focused
- Reference issues/tasks in commit messages
- Use meaningful branch names

**Documentation references:**
- [Ansible style guide](https://docs.ansible.com/ansible/latest/dev_guide/style_guide/)
- [Docker Compose best practices](https://docs.docker.com/compose/production/)
- [Conventional Commits](https://www.conventionalcommits.org/)

---

## Project-Specific Context

### Technology Stack

| Component | Version/Details | Documentation |
|-----------|----------------|---------------|
| **OS** | Raspberry Pi OS Lite (64-bit) Bookworm | [Docs](https://www.raspberrypi.com/documentation/) |
| **Ansible** | Latest (running in devcontainer) | [Docs](https://docs.ansible.com/) |
| **OpenMediaVault** | v7.x | [Docs](https://docs.openmediavault.org/) |
| **Docker** | Installed via OMV-Extras | [Docs](https://docs.docker.com/) |
| **Immich** | Self-hosted photo management | [Docs](https://immich.app/docs) |
| **Hardware** | Raspberry Pi 5 (4GB/8GB RAM) | [Docs](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html) |

### Project Structure

```
ansible/
├── ansible.cfg           # Ansible configuration
├── inventory/
│   ├── hosts.yml         # Inventory (git-ignored, user-specific)
│   └── hosts.yml.example # Template for inventory
├── playbooks/
│   ├── site.yml          # Master playbook (runs all)
│   ├── omv.yml           # OpenMediaVault setup
│   └── immich.yml        # Immich deployment
└── roles/
    ├── omv/              # OpenMediaVault role
    │   ├── defaults/     # Default variables
    │   ├── handlers/     # Service handlers
    │   ├── tasks/        # Task files (modular)
    │   └── templates/    # Jinja2 templates
    └── immich/           # Immich role
        ├── defaults/
        ├── handlers/
        ├── tasks/
        └── templates/
            ├── docker-compose.yml.j2
            └── immich.env.j2
```

### Current State & Roadmap

- ✅ **Completed**: Base Ansible automation, OMV installation, Immich deployment
- 🚧 **In Progress**: See `docs/roadmap.md` for planned enhancements
- 📋 **Backlog**: External SSD storage, boot from SSD, encrypted cloud backups

### Common Tasks

**Running playbooks:**
```bash
cd ansible
ansible-playbook playbooks/site.yml           # Full setup
ansible-playbook playbooks/omv.yml            # Only OMV
ansible-playbook playbooks/immich.yml         # Only Immich
ansible-playbook playbooks/site.yml --check   # Dry run
ansible-playbook playbooks/site.yml --tags=docker  # Specific tags
```

**Testing idempotency:**
```bash
ansible-playbook playbooks/site.yml           # First run (changes expected)
ansible-playbook playbooks/site.yml           # Second run (should be green)
```

**Linting:**
```bash
ansible-lint ansible/playbooks/
```

---

## Code Review Checklist

When reviewing or generating code, ensure:

- [ ] **Idempotent**: Can be run multiple times safely
- [ ] **Secure**: No hardcoded secrets, follows least privilege
- [ ] **Documented**: Clear task names, comments where needed, README updates
- [ ] **Tested**: Considered edge cases, tested in check mode
- [ ] **Versioned**: Docker images pinned, dependencies specified
- [ ] **Error handling**: Failures handled gracefully with meaningful messages
- [ ] **Tagged**: Appropriate tags for selective execution
- [ ] **Variables**: Properly scoped, sensible defaults, documented
- [ ] **Handlers**: Services restarted via handlers, not inline
- [ ] **Compatibility**: Works with Raspberry Pi OS Bookworm and target versions

---

## Questions to Always Consider

Before implementing any feature or fix:

1. **Is this the Ansible way?** (Is there a module instead of shell?)
2. **What happens on the second run?** (Idempotency check)
3. **What if it fails halfway?** (Can it recover?)
4. **Who can access this?** (Security implications)
5. **Where is this documented?** (User-facing docs)
6. **Can this be tested in check mode?** (Dry run validation)
7. **What are the dependencies?** (Ordering, prerequisites)
8. **Is there a better practice?** (Industry standards)

---

## Additional Resources

- **Ansible Collections in Use:**
  - `community.general` - [Docs](https://docs.ansible.com/ansible/latest/collections/community/general/)
  - `ansible.posix` - [Docs](https://docs.ansible.com/ansible/latest/collections/ansible/posix/)
  
- **Project Documentation:**
  - `README.md` - Quick start guide
  - `docs/setup.md` - Detailed setup instructions
  - `docs/roadmap.md` - Feature roadmap and backlog

- **External Documentation:**
  - [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)
  - [OpenMediaVault Forums](https://forum.openmediavault.org/)
  - [Immich Documentation](https://immich.app/docs)
  - [Ansible Lint](https://ansible.readthedocs.io/projects/lint/)

---

## Summary

When working on this project:
1. **Security first** - vault secrets, least privilege, secure defaults
2. **Idempotency always** - test by running twice
3. **Document everything** - code comments, READMEs, user docs
4. **Ask questions** - challenge assumptions, validate approach
5. **Follow standards** - Ansible best practices, industry conventions
6. **Test thoroughly** - check mode, multiple runs, edge cases
7. **Keep it simple** - readable, maintainable, well-structured

Remember: This project manages critical home infrastructure. Reliability, security, and maintainability are paramount.
