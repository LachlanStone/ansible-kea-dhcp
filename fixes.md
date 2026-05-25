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

## Issue #6 — `keactrl.conf.j2` has hardcoded service state

**Status:** 🔴 Open

**Problem:**
`keactrl.conf.j2` hardcodes `dhcp4=yes`, `dhcp6=yes`, `dhcp_ddns=no` regardless of the `kea_dhcp_service_state` variable. A user who disables IPv4 or enables IPv6 via `kea_dhcp_service_state` will get a config that contradicts their intent.

**What needs to be done:**
- Render `dhcp4`, `dhcp6`, and `dhcp_ddns` values from `kea_dhcp_service_state.ipv4`, `kea_dhcp_service_state.ipv6`, and `kea_dhcp_service_state.ddns` respectively.

---

## Issue #7 — `keactrl.conf` template is never deployed

**Status:** 🔴 Open

**Problem:**
`templates/etc/kea/keactrl.conf.j2` exists but `tasks/config.yml` never includes it in the template loop. The file is never written to the target host.

**What needs to be done:**
- Add `etc/kea/keactrl.conf` to the loop in `tasks/config.yml` (after fixing Issue #6 above).

---

## Issue #8 — No `kea-dhcp6.conf.j2` template

**Status:** 🔴 Open

**Problem:**
The role manages the `kea-dhcp6-server` service (enabled/disabled via `kea_dhcp_service_state.ipv6`) but there is no Jinja2 template for `/etc/kea/kea-dhcp6.conf`, and no corresponding default variable. IPv6 cannot actually be configured through this role.

**What needs to be done:**
- Add `kea_dhcp_dhcp6_config` default variable in `defaults/main.yml`.
- Create `templates/etc/kea/kea-dhcp6.conf.j2`.
- Add it to `tasks/config.yml` loop, gated on `kea_dhcp_service_state.ipv6`.

---

## Issue #9 — Disabled services are not stopped

**Status:** 🔴 Open

**Problem:**
`tasks/services.yml` only starts services where `enabled: true`. If a service was previously running and the user sets it to `enabled: false`, the role disables it from boot but does not stop the currently running process.

**What needs to be done:**
- Add a task that stops services where `item['enabled'] | bool` is false.

---

## Issue #10 — Verbose `hostvars[inventory_hostname]` pattern

**Status:** 🔴 Open

**Problem:**
`tasks/redhat.yml` uses `hostvars[inventory_hostname].ansible_distribution_major_version` instead of the simpler `ansible_distribution_major_version`. These are equivalent but the former is unnecessarily verbose and harder to read.

**What needs to be done:**
- Replace all occurrences of `hostvars[inventory_hostname].ansible_*` with the direct magic variable.

---

## Issue #11 — `requirements.txt` is unpinned and stale

**Status:** 🔴 Open

**Problem:**
`requirements.txt` lists `testinfra` and `flake8` which are no longer the standard for molecule-based testing. The modern stack uses `molecule-plugins[docker]`. Nothing is pinned, meaning installs are non-reproducible.

**What needs to be done:**
- Replace contents with a pinned set: `ansible`, `ansible-lint`, `molecule`, `molecule-plugins[docker]`, `docker`.
- Remove `testinfra` and `flake8` (not used anywhere in the current test setup).
- Consider adding a `requirements-dev.txt` with pinned versions for local development.

---

## Issue #12 — SUSE tasks have no repository setup

**Status:** 🔴 Open

**Problem:**
`tasks/suse.yml` goes straight to package installation with no repository configuration. On a clean SUSE/openSUSE system, Kea packages will not be in the default repos and the install will fail.

**What needs to be done:**
- Add repo configuration for SUSE/openSUSE (ISC Cloudsmith provides SUSE repos).
- Add repo key handling.
- Add corresponding `set_facts.yml` entries for SUSE repo URLs.

---

## Issue #13 — `kea.conf.j2` uses a fragile dict-update pattern

**Status:** 🔴 Open

**Problem:**
`templates/etc/kea/kea.conf.j2` uses the pattern:
```jinja2
{% set kea = {} %}
{% set _kea = kea.update(kea_dhcp_dhcp4_config) %}
{% set _kea = kea.update(kea_dhcp_ddns_config) %}
{{ kea|to_nice_json }}
```
The `dict.update()` method in Jinja2 returns `None`, so `_kea` is always `None`. The side-effect on `kea` works in CPython but this relies on mutable dict behaviour that is not guaranteed consistent across Jinja2 versions. Additionally if both configs have overlapping top-level keys the second will silently overwrite the first.

**What needs to be done:**
- Replace with `combine` filter: `{{ kea_dhcp_dhcp4_config | combine(kea_dhcp_ddns_config) | to_nice_json }}`.
- Validate that key namespaces don't collide between `Dhcp4`/`DhcpDdns` and `Logging`.
