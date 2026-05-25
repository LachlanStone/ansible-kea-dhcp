# Role Fix Tracker

This document tracks all identified issues with the `ansible-kea-dhcp` role, including what was changed and why. Issues are resolved incrementally — see each entry for current status.

---

## Issue #1 — Kea version is severely outdated

**Status:** 🔴 Open

**Problem:**
`kea_dhcp_version: 1.7` is set as the default. Kea is now at **2.6.x** as of 2024 and v1.7 is end-of-life. The entire `set_facts.yml` and `redhat.yml` repo logic only handles versions 1.6 and 1.7, meaning the role cannot install any modern version of Kea on any supported OS.

**What needs to be done:**
- Update `kea_dhcp_version` default to a current 2.x release.
- Add `set_facts.yml` blocks for each supported 2.x release line.
- Update repo URLs for both Debian and RedHat families to point to current ISC Cloudsmith repos.
- Review and update package names (they changed in 2.x for some distros).

---

## Issue #2 — Kea 2.x config format changed

**Status:** 🔴 Open

**Problem:**
In Kea 2.x the `Logging` configuration block was integrated into each daemon config directly — it is no longer a separate top-level key alongside `Dhcp4`/`DhcpDdns`. The current `defaults/main.yml` structure produces invalid JSON config on Kea 2.x installs.

Additionally, `keactrl` was soft-deprecated in Kea 2.x in favour of the **Kea Control Agent** (`kea-ctrl-agent`). The `keactrl.conf.j2` template still reflects the old model.

**What needs to be done:**
- Restructure `kea_dhcp_dhcp4_config` and `kea_dhcp_ddns_config` in `defaults/main.yml` to move logging inside the daemon object (Kea 2.x format).
- Add a `kea_dhcp_ctrl_agent_config` default and matching template for `kea-ctrl-agent`.
- Update or deprecate `keactrl.conf.j2`.

---

## Issue #3 — `apt_key` is deprecated ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`ansible.builtin.apt_key` adds GPG keys to a legacy shared keyring (`/etc/apt/trusted.gpg` or `/etc/apt/trusted.gpg.d/`). This approach was deprecated in Debian 11+ and Ubuntu 22.04+ because a single compromised key could affect all repositories. Modern best practice is to store per-repo keys under `/etc/apt/keyrings/` and reference them via `signed-by=` in the apt source line.

**What was changed:**

`tasks/debian.yml`:
- Removed the `ansible.builtin.apt_key` task.
- Added an `ansible.builtin.file` task to ensure `/etc/apt/keyrings/` exists with correct permissions.
- Replaced with `ansible.builtin.get_url` to download the key directly to `/etc/apt/keyrings/isc-kea.asc`.

`tasks/set_facts.yml`:
- Updated all Debian apt repository strings to include `[signed-by=/etc/apt/keyrings/isc-kea.asc]` in the `deb` line so apt knows which key to use for this specific repo.

**Why this matters:**
Without `signed-by=`, modern apt will warn or refuse to use the repo on Debian 12 / Ubuntu 22.04+. This fix brings the role into alignment with current Debian/Ubuntu security best practice.

---

## Issue #4 — All molecule scenarios target EOL distros ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
Every existing molecule scenario targeted an end-of-life distribution:
- `centos7` — EOL June 2024
- `centos8` — EOL December 2021
- `debian8`, `debian9`, `debian10` — all EOL
- `ubuntu1604`, `ubuntu1804` — both EOL

As a result, the molecule tests in GitHub Actions were completely commented out with a TODO note. The role had zero automated integration testing running in CI.

**What was changed:**
- Created four new molecule scenarios targeting current, supported distributions:
  - `debian12` — Debian 12 Bookworm (current stable)
  - `ubuntu2204` — Ubuntu 22.04 LTS Jammy
  - `ubuntu2404` — Ubuntu 24.04 LTS Noble
  - `rocky9` — Rocky Linux 9 (RHEL 9 compatible)
- Updated `.github/workflows/default.yml` to uncomment and enable the molecule job using the new scenarios.
- Rocky 9 uses `geerlingguy/docker-rockylinux9-ansible` (the community standard image for RHEL-family Ansible testing). Debian and Ubuntu scenarios continue using `jrei/systemd-*` images, consistent with the existing scenario style.

**Note:** Full end-to-end molecule test success is also blocked by Issue #1 (outdated Kea version) and Issue #3 (deprecated apt_key). Issue #3 is resolved above. Issue #1 still needs Kea 2.x support added before installs will succeed on these new distros.

---

## Issue #5 — `.travis.yml` is dead code ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`.travis.yml` still existed in the repo root. The project migrated to GitHub Actions (`.github/workflows/`) and Travis CI is no longer used. The file was referencing EOL distro scenarios (`centos7`, `debian9`, `debian10`, `ubuntu1604`, `ubuntu1804`) and would not have worked even if Travis CI was still active.

**What was changed:**
- Deleted `.travis.yml`.

**Why this matters:**
Dead CI config files create confusion about how the project is tested, and the old Travis notification webhook pointed at Galaxy v1 API which no longer exists.

---

## Issue #6 — `keactrl.conf.j2` has hardcoded service state ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`keactrl.conf.j2` hardcodes `dhcp4=yes`, `dhcp6=yes`, `dhcp_ddns=no` regardless of the `kea_dhcp_service_state` variable. A user who disables IPv4 or enables IPv6 via `kea_dhcp_service_state` will get a config that contradicts their intent.

**What was changed:**
- `templates/etc/kea/keactrl.conf.j2`: Replaced hardcoded `yes`/`no` with Jinja2 conditionals:
  - `dhcp4={{ 'yes' if kea_dhcp_service_state['ipv4'] | bool else 'no' }}`
  - `dhcp6={{ 'yes' if kea_dhcp_service_state['ipv6'] | bool else 'no' }}`
  - `dhcp_ddns={{ 'yes' if kea_dhcp_service_state['ddns'] | bool else 'no' }}`

**Why this matters:**
Previously the keactrl config was static and always started both DHCPv4 and DHCPv6 regardless of what the user configured. Now it accurately reflects the intended service state.

---

## Issue #7 — `keactrl.conf` template is never deployed ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`templates/etc/kea/keactrl.conf.j2` existed but was never included in the `tasks/config.yml` template loop, so it was never written to the target host.

**What was changed:**
- `tasks/config.yml`: Added `etc/kea/keactrl.conf` to the template deploy loop alongside the existing DDNS and DHCPv4 configs.

**Why this matters:**
Without deploying this file, any customisation in `keactrl.conf.j2` (including the fix from Issue #6) would never take effect on the managed host.

---

## Issue #8 — No `kea-dhcp6.conf.j2` template ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
The role managed the `kea-dhcp6-server` service but there was no Jinja2 template for `/etc/kea/kea-dhcp6.conf` and no corresponding default variable. IPv6 DHCP could not actually be configured through this role.

**What was changed:**
- `defaults/main.yml`: Added `kea_dhcp_dhcp6_config` default variable with sensible baseline DHCPv6 settings (timers, memfile lease database, empty `subnet6` list, logging).
- `templates/etc/kea/kea-dhcp6.conf.j2`: Created new template that renders `kea_dhcp_dhcp6_config` to JSON.
- `tasks/config.yml`: Added a dedicated task to deploy `/etc/kea/kea-dhcp6.conf` from the template, gated on `kea_dhcp_service_state['ipv6'] | bool` so it only runs when IPv6 is actually enabled.

**Why this matters:**
Previously the role had no way to produce a valid DHCPv6 config. The service would start (if enabled) but fail immediately without a config file.

---

## Issue #9 — Disabled services are not stopped ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`tasks/services.yml` only started services where `enabled: true`. If a service was previously running and the user set it to `enabled: false`, the role would disable it from boot but leave the currently running process alive.

**What was changed:**
- `tasks/services.yml`: Added a third task "Stopping Disabled Kea DHCP Services" that issues `state: stopped` for any service where `item['enabled'] | bool` is false.

**Why this matters:**
Without this fix, disabling a service through Ansible would only take effect after the next reboot. With this fix, the actual running state of the service is immediately reconciled with the desired state.

---

## Issue #10 — Verbose `hostvars[inventory_hostname]` pattern ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`tasks/redhat.yml` used `hostvars[inventory_hostname].ansible_distribution_major_version` in all three `yum_repository` `baseurl` values. This is functionally equivalent to the direct magic variable but unnecessarily verbose and harder to read.

**What was changed:**
- `tasks/redhat.yml`: Replaced all three occurrences of `hostvars[inventory_hostname].ansible_distribution_major_version` with `ansible_distribution_major_version`.

**Why this matters:**
The verbose form is a common anti-pattern that makes playbooks harder to read and maintain. The direct magic variable is the Ansible-recommended approach.

---

## Issue #11 — `requirements.txt` is unpinned and stale ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`requirements.txt` listed `testinfra` and `flake8` which are not part of the current molecule testing stack. `testinfra` was the old verifier (replaced by Ansible-native verify playbooks). `flake8` was for Python linting not used anywhere in the role. Nothing was pinned, making installs non-reproducible.

**What was changed:**
- `requirements.txt`: Removed `flake8` and `testinfra`. Added `molecule-plugins[docker]` which is the current way to install the Docker driver for Molecule.

**Why this matters:**
`molecule[docker]` (old style) no longer works — the Docker driver now ships separately as `molecule-plugins[docker]`. Installing the old package name would result in a broken molecule setup.

---

## Issue #12 — SUSE tasks have no repository setup ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`tasks/suse.yml` went straight to package installation with no repository configuration. On a clean SUSE/openSUSE system, Kea packages are not in the default repos and the install would fail.

**What was changed:**
- `tasks/set_facts.yml`: Added a new `set_facts | Setting Repo Info (Suse)` task that sets `kea_dhcp_suse_repo_url` (pointing at the ISC Cloudsmith SUSE-compatible RPM repo) and `kea_dhcp_repo_key` when `ansible_os_family == "Suse"`.
- `tasks/suse.yml`: Added two tasks before the package install:
  1. `ansible.builtin.rpm_key` — imports the ISC GPG key.
  2. `community.general.zypper_repository` — adds the ISC Cloudsmith repo with `auto_import_keys: true`.

**Why this matters:**
Without repo configuration, the SUSE path was completely non-functional on any clean host. This brings it to parity with the Debian and RedHat paths.

**Note:** `community.general.zypper_repository` requires the `community.general` collection, which should already be available via `ansible` package installs >= 4.0.

---

## Issue #13 — `kea.conf.j2` uses a fragile dict-update pattern ✅ Fixed

**Status:** ✅ Resolved

**Problem:**
`templates/etc/kea/kea.conf.j2` used:
```jinja2
{% set kea = {} %}
{% set _kea = kea.update(kea_dhcp_dhcp4_config) %}
{% set _kea = kea.update(kea_dhcp_ddns_config) %}
{{ kea|to_nice_json }}
```
The `dict.update()` method in Jinja2 returns `None`, so `_kea` is always `None`. The side-effect mutation of `kea` works in CPython but is not guaranteed behaviour across Jinja2 versions. If the two config dicts share a top-level key (e.g., both had `Logging`), the second silently overwrites the first.

**What was changed:**
- `templates/etc/kea/kea.conf.j2`: Replaced with:
  ```jinja2
  {{ kea_dhcp_dhcp4_config | combine(kea_dhcp_ddns_config) | to_nice_json }}
  ```

**Why this matters:**
`combine` is the Ansible-idiomatic, well-tested way to merge dictionaries in Jinja2. It is reliable across all supported Ansible/Jinja2 versions and its behaviour with key conflicts is well-documented (last-wins by default, or recursive with `recursive=True`).
