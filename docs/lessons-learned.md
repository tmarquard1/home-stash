# Lessons Learned: SSD Storage + OMV Integration

This document captures hard-won knowledge from setting up external SSD storage
with OpenMediaVault and Immich via Ansible automation. Each section describes a
problem, why it happened, and how to avoid it.

---

## 1. OMV Has Its Own Config Database — Never Bypass It

**What happened:** We mounted the SSD using standard Linux tools (`/etc/fstab`,
`mount -a`) which worked perfectly at the OS level, but OMV's web UI never
showed the filesystem under Storage > File Systems.

**Why:** OMV maintains its own XML configuration database
(`/etc/openmediavault/config.xml`). The web UI reads from this database, not
from `/etc/fstab` directly. OMV's salt-based deployment system *generates*
`/etc/fstab` from its database — not the other way around.

**The fix:** Register filesystems using OMV's RPC API:
```bash
omv-rpc -u admin "FsTab" "set" '<json config>'
```

**Rule:** If OMV manages a service, always use OMV's API/RPC to configure it.
Direct system changes will be invisible to the UI and may be overwritten.

**Key commands for debugging:**
```bash
# Read OMV's mount point database
omv-confdbadm read conf.system.filesystem.mountpoint

# List all filesystems OMV can see (OS-level detection)
omv-rpc -u admin "FileSystemMgmt" "enumerateFilesystems"

# Check if config is dirty (pending changes)
omv-rpc -u admin "Config" "isDirty" '{"id":""}'
```

---

## 2. OMV's Sentinel UUID for New Config Objects

**What happened:** We tried to register a new mount point using a random UUID.
OMV's `FsTab.set()` RPC interpreted it as an *update* to an existing entry,
failed to find it, and threw an XPath query error.

**Why:** OMV uses a special sentinel UUID to distinguish "create new" from
"update existing":
```
fa4b1c66-ef79-11e5-87a0-0002b3a176b4
```

This value is defined in `/etc/default/openmediavault` as
`OMV_CONFIGOBJECT_NEW_UUID`. When the RPC receives this UUID, it auto-generates
a real UUID and creates a new database entry.

**Rule:** Always use the sentinel UUID when creating new OMV config objects via
RPC. This is stable across all OMV installations (same since OMV 3.x).

---

## 3. Don't Hardcode Partition Numbers

**What happened:** The Crucial X9 SSD ships with a factory partition layout
(small EFI partition + large exFAT data partition as partition 2). We hardcoded
`omv_external_disk_partition: "2"` to target the data partition, but this
was fragile and specific to one disk model.

**Why:** Different SSDs ship with different partition layouts. After
reformatting, the partition number changes (our GPT creates partition 1).

**The fix:** Use filesystem label detection (`blkid -L <label>`) to find the
correct partition regardless of its number. On first run, wipe the entire disk
and create a clean single-partition layout.

**Rule:** Never hardcode partition numbers. Use labels or UUIDs for detection.

---

## 4. The `force: true` Trap on Filesystem Formatting

**What happened:** To handle the factory exFAT filesystem, we set `force: true`
on the `community.general.filesystem` task. This is dangerous because it will
reformat even if there's valid data on the partition.

**Why:** The filesystem module with `force: true` will overwrite any existing
filesystem, including one with user data.

**The fix:** Separate the logic:
1. Check if filesystem with expected label already exists → skip formatting
2. If no matching filesystem → wipe disk completely, repartition, format fresh

**Rule:** Never use `force: true` on filesystem formatting. Use explicit
detection logic to decide when formatting is safe.

---

## 5. Docker Package Conflicts on Debian Bookworm/Trixie

**What happened:** OMV's compose module tried to install `docker-compose-plugin`
(from Docker's official repo), but Debian's `docker-compose` package was already
installed. Both packages provide the same file
(`/usr/libexec/docker/cli-plugins/docker-compose`), causing a dpkg conflict.

**Result:** The `compose` salt module failed, which made `Config.applyChanges`
return a 500 error, which left the "pending configuration changes" banner stuck.

**The fix:** Remove the Debian `docker-compose` package before the OMV compose
module tries to install `docker-compose-plugin`:
```bash
sudo apt remove -y docker-compose
```

**Rule:** On fresh installs, the Ansible playbook should remove conflicting
Debian Docker packages before OMV-Extras installs its preferred versions. Add
this to the `install_extras.yml` task file.

**Affected packages to watch for:**
- `docker-compose` (Debian) vs `docker-compose-plugin` (Docker repo)
- `docker.io` (Debian) vs `docker-ce` (Docker repo)
- `containerd` (Debian) vs `containerd.io` (Docker repo)

---

## 6. `omv-salt deploy run` Requires Module Names

**What happened:** We ran `omv-salt deploy run --no-color` without specifying
which module to deploy. It returned an error:
```
Error: Missing argument '[NAMES]...'
```

**Why:** Unlike the web UI's "Apply" button (which applies all pending modules),
the `omv-salt` CLI requires explicit module names.

**The fix:** Either specify modules explicitly:
```bash
omv-salt deploy run fstab
omv-salt deploy run collectd cron monit
```
Or use the RPC to apply all pending changes (equivalent to clicking "Apply"):
```bash
omv-rpc -u admin "Config" "applyChanges" '{"modules":[], "force":false}'
```

**Rule:** Use `omv-rpc Config.applyChanges` for "apply all". Use
`omv-salt deploy run <module>` when targeting specific modules.

---

## 7. `Config.applyChanges` Can Trigger Service Restarts / Reboots

**What happened:** Running `Config.applyChanges` via RPC caused the Pi to
become unreachable (SSH connection dropped). It appeared to reboot or restart
networking.

**Why:** Applying pending OMV configuration can restart services like SSH,
networking, Docker, or even trigger a reboot depending on what changed. This is
the same as clicking "Apply" in the web UI.

**Rule:**
- Expect SSH disconnection when applying OMV config changes remotely
- In Ansible, use `async` + `poll` for the apply task, or add
  `wait_for_connection` after it
- Don't apply changes at the end of a long SSH session where losing the
  connection is costly

---

## 8. Test Idempotency by Running Twice — But on a Clean System

**What happened:** We tested idempotency on a system that already had manual
fixes applied (manually mounted SSD, manually edited fstab). The playbook
appeared idempotent, but it was actually just skipping work because the end
state already existed — not because it could *create* that state from scratch.

**Rule:** True idempotency testing requires:
1. Run on a clean system → should make changes
2. Run again immediately → should report `changed=0`
3. Wipe the specific state and run again → should recreate it

---

## 9. OMV RPC Reference

**Useful RPC calls discovered during this work:**

| Purpose | Command |
|---------|---------|
| List all filesystems | `omv-rpc -u admin "FileSystemMgmt" "enumerateFilesystems"` |
| Create mount point | `omv-rpc -u admin "FsTab" "set" '<json>'` |
| Read config database | `omv-confdbadm read conf.system.filesystem.mountpoint` |
| Check pending changes | `omv-rpc -u admin "Config" "isDirty" '{"id":""}'` |
| Apply all changes | `omv-rpc -u admin "Config" "applyChanges" '{"modules":[],"force":false}'` |
| Deploy specific module | `omv-salt deploy run <module_name>` |

**OMV config data model for mount points** is at:
```
/usr/share/openmediavault/datamodels/conf.system.filesystem.mountpoint.json
```

**OMV environment variables** (including the sentinel UUID):
```
/etc/default/openmediavault
```

---

## 10. Known Issues Still Open

1. **Pending configuration changes banner** — May still show after Ansible runs
   if modules like `collectd`, `monit`, `cron` haven't been deployed since OMV
   install. The final `Config.applyChanges` RPC in `main.yml` should handle
   this, but watch for SSH drops.

2. **docker-compose package conflict** — Not yet handled in Ansible. A fresh
   install will hit this. Need to add a task to remove `docker-compose` (Debian)
   before OMV-Extras installs `docker-compose-plugin`.

3. **`install_extras.yml` deploys only nginx** — The old code ran
   `omv-salt deploy run nginx` which only deployed one module. The new code
   removed this in favor of the end-of-role `Config.applyChanges`, but this
   needs testing on a fresh install.

---

## Summary: What To Do Differently Next Time

1. **Read OMV source code first** — OMV's RPC/salt internals aren't well
   documented. Check `/usr/share/openmediavault/engined/rpc/` and
   `/usr/share/openmediavault/datamodels/` on the Pi for the actual API.

2. **Don't mix OMV management with raw Linux commands** — If OMV manages
   something (fstab, Docker, services), go through OMV's API.

3. **Test on a truly fresh system** — Not on one with accumulated manual fixes.

4. **Handle the docker package conflict proactively** — Remove Debian Docker
   packages before installing OMV-Extras.

5. **Expect the unexpected with `Config.applyChanges`** — It can restart
   services and drop SSH connections. Plan for it.

6. **Use labels, not partition numbers** — For any disk detection logic.

7. **Keep the playbook simple** — Avoid migration code and complex conditional
   logic. A clean wipe-and-setup approach is more reliable than trying to handle
   every possible existing state.
