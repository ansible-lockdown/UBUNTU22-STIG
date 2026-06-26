# Molecule Quick Start - Ubuntu 22.04 LTS STIG

> Audience: anyone new to molecule testing with this Ansible Lockdown role. Read top-to-bottom the first time; come back to the reference tables later.

## What this scenario does

This `molecule/` directory ships one scenario: `default/`. It spins up a throwaway Ubuntu 22.04 LTS Docker container, applies the **whole** Ansible Lockdown Ubuntu 22.04 STIG role against it, then runs the paired goss audit (from the `UBUNTU22-STIG-Audit` repo) to see how many controls passed and how many still fail. It's the gating test you run before merging or pushing benchmark changes.

End-to-end you get:

1. A clean container booted into systemd.
2. Pre-remediation audit (baseline): records which controls are already failing before the role runs.
3. Role converge #1: applies all the STIG remediations.
4. Role converge #2: re-runs to confirm idempotency (zero changed tasks the second time).
5. Post-remediation audit: records the after-picture.
6. Pre and post audit JSON summaries automatically copied out to a directory beside the role for review.

If steps 3 and 4 both report `failed=0` and step 6 shows the post-audit failure count substantially lower than the pre-audit count, the role is shippable.

## Prerequisites

| Need | Why | How to check |
|---|---|---|
| Docker (Desktop or Engine), running | Molecule uses Docker to create the test container | `docker info` returns without error |
| Python 3.10+ | Ansible / Molecule are Python tools | `python3 --version` |
| Ansible venv with `ansible-core >= 2.16.1`, `molecule`, `molecule-plugins[docker]`, `docker`, `passlib` | Runtime deps for the test (matches `meta/main.yml` `min_ansible_version`) | `pip list \| grep -E 'ansible\|molecule'` |
| `git` on the controller | Audit content is cloned from `UBUNTU22-STIG-Audit` | `git --version` |
| Local clones of **both** `UBUNTU22-STIG` AND `UBUNTU22-STIG-Audit` (or network access so the audit repo can be cloned at runtime) | The role pulls audit goss content from the audit repo during converge | `git -C <path-to>/UBUNTU22-STIG-Audit status` |

One-time venv setup (skip if you already have one):

```bash
python3 -m venv <path-to-your-ansible-venv>
source <path-to-your-ansible-venv>/bin/activate
pip install 'ansible-core>=2.16.1' 'molecule>=24' 'molecule-plugins[docker]' docker passlib
```

## Quick start

```bash
# Every time
source <path-to-your-ansible-venv>/bin/activate
cd <path-to>/UBUNTU22-STIG

# Full gating pair
molecule destroy && molecule converge && molecule converge && molecule verify
```

The whole sequence takes 5-15 minutes on a modern laptop (smaller package set and lighter container than the RHEL family). Network is needed (clone of audit content + apt installs) for the first run; subsequent runs reuse the cached Docker image.

## What each command does

| Command | What happens |
|---|---|
| `molecule destroy` | Removes any leftover container from a previous run. Safe to run on a clean system. |
| `molecule converge` (first) | (a) pulls / starts the `geerlingguy/docker-ubuntu2204-ansible:latest` container, (b) runs `prepare.yml` to install supporting packages, (c) clones the audit content from `UBUNTU22-STIG-Audit`, (d) runs the pre-remediation audit, (e) applies the full role, (f) runs the post-remediation audit, (g) fetches both audit JSONs to your local `_temp_fetched_audits/` folder. |
| `molecule converge` (second) | Re-runs the role against the already-remediated container. The Ansible `PLAY RECAP` should show `changed=0` (or a small handful of acceptable re-renders) - that's the **idempotency check**. |
| `molecule verify` | Final assertions defined in the scenario (light by default for this role; the real validation is in the audit JSONs). |

## Where the audit results land

After `molecule converge` completes, look at:

```
<working-tree>/_temp_fetched_audits/
  ubuntu2204-UBUNTU22-STIG-v<ver>_pre_scan_<timestamp>.json
  ubuntu2204-UBUNTU22-STIG-v<ver>_post_scan_<timestamp>.json
```

The JSONs are goss output - each has a `results: []` array where every entry has a `successful: true|false` field. Compare pre vs. post:

```bash
# How many controls passed before remediation?
jq '[.results[] | select(.successful==true)] | length' <pre_scan>.json

# How many passed after?
jq '[.results[] | select(.successful==true)] | length' <post_scan>.json
```

If the second number is significantly higher than the first, the role is working as intended.

## How to read `PLAY RECAP`

After each converge, Ansible prints a one-line summary like:

```
PLAY RECAP ********************************************************
ubuntu2204 : ok=171  changed=55  unreachable=0  failed=0  skipped=233  rescued=0  ignored=0
```

| Counter | What it means |
|---|---|
| `ok` | Tasks that ran and finished successfully (no state change needed, or already in desired state) |
| `changed` | Tasks that modified the system (installed a package, edited a file, etc.) |
| `failed` | Tasks that errored out. **MUST BE 0** for the run to be considered passing. |
| `skipped` | Tasks gated off by `when:` conditions (often because `system_is_container: true` skips container-incompatible work) |
| `rescued` / `ignored` | Block-level error handling - typically 0 |

**Idempotency expectation:** the second `converge` should show `changed=0` (or near zero) - the system was already in the desired state. A high `changed` count on the second run means a task is not idempotent and needs fixing.

The pre/post audit shell tasks and the `Create ansible facts file` task always re-render (they're actions, not state changes); 2-3 `changed` on the second converge driven by those is acceptable.

## Why some audit failures are expected (even after the role passes)

The post-scan will still show some controls failing. Most fall into three buckets that aren't role bugs - they're inherent to containerized testing:

1. **SSH-config controls** (banner, ciphers, KexAlgorithms, MACs, idle timeouts, login grace time): on a fresh Ubuntu container `sshd` isn't running (we install `openssh-server` but don't start the daemon - containers use `docker exec` for shell access). Goss checks for active `/etc/ssh/sshd_config` enforcement; on real hosts these controls pass after a `Restart sshd` handler fires.

2. **User-state controls** (password complexity history, password lifetime, dotfile audit): need real interactive users (UID >= 1000) with shadow entries and home directories. Containers only have system accounts, so the controls can't satisfy their preconditions.

3. **Kernel / mount / boot controls**: any control gated by `not system_is_container` is intentionally skipped during remediation (auditd kernel rules, AppArmor profiles backed by kernel modules, kernel sysctls, mount options, GRUB / boot CMDLINE) because containers share the host kernel. Audit still runs the goss tests, so they report as failing.

None of these warrant fixing the role. They're the expected delta between a containerized test and a real Ubuntu 22.04 host.

## Common gotchas

| Symptom | Likely cause | Fix |
|---|---|---|
| `molecule converge` errors with "container not running" | Stale container from a previous interrupted run | Run `molecule destroy` first, then retry |
| Pre-task fails with "Unable to acquire the dpkg frontend lock" | Another apt process is running (cron, unattended-upgrades) inside the prepared container | Re-run; if persistent, add a 30s sleep at the top of `prepare.yml` |
| `/bin/sh: 1: set: Illegal option -o pipefail`, or a piped shell behaving oddly (Ubuntu `/bin/sh` is dash) | A `shell:` task running under dash instead of bash | The role forces bash on every `shell:` task via `args: executable: "{{ ubtu22stig_shell_executable }}"` (default `/bin/bash`, set in `defaults/main.yml`), so this should not occur. If you add a new shell task, include that same `args:` block; grep `tasks/` for any `ansible.builtin.shell` without it. |
| Converge #2 fails on a task that passed in #1 | Ansible 2.19+ struct-vs-string type-check tripping a `when:` clause that's "lucky" on the first run | Inspect the offending task's `when:` - look for quoted-string-as-boolean bugs |
| `Conditional result (True) was derived from value of type 'str'` | Ansible 2.19+ rejects when-clauses whose final value is a non-boolean string. Common bug: an `or "<some expression>"` where the RHS got wrapped in quotes by accident | Remove the quotes; the RHS should be a bare Jinja expression |
| `verify` step empty / passes trivially | Expected - this role's primary validation is the goss audit JSONs, not molecule's `verifier:` plays | Inspect the audit JSONs instead |

## Apple Silicon (M-series Mac) note

`geerlingguy/docker-ubuntu2204-ansible:latest` is a multi-arch image (amd64 + arm64), so on Apple Silicon it pulls the arm64 variant natively - no Rosetta required. The scenario leaves `platform:` unspecified to take advantage of that.

If you specifically need to reproduce a CI runner's behaviour (which is amd64), add `platform: linux/amd64` to the platform entry in `molecule.yml`. That path requires Docker Desktop's "Use Rosetta for x86_64/amd64 emulation" toggle and runs noticeably slower.

## Reference: what's in the default scenario

| File | Purpose |
|---|---|
| `molecule.yml` | Driver: docker. Image: `geerlingguy/docker-ubuntu2204-ansible:latest` (multi-arch; native arm64 on Apple Silicon, amd64 elsewhere). systemd as PID 1 via `command: /lib/systemd/systemd`. Host vars listed in the next table. |
| `prepare.yml` | Installs supporting packages (`openssh-server`, `libpam-pwquality`, `libpam-modules`, `sudo`, `acl`, `kmod`, `cron`, `chrony`, `rsyslog`, `aide`, `aide-common`, `logrotate`, `apparmor`, `apparmor-utils`, `ufw`, `nftables`, `auditd`, `autofs`, `python3-apt`). Creates `/run/sshd`, `/etc/sysctl.d`, `/etc/ssh/sshd_config.d`. Starts `cron` and `rsyslog`. |
| `converge.yml` | Sets root password (so PAM tasks don't lock us out), stubs `/etc/default/grub` so GRUB-tag tasks don't fail in the container, includes the role. |

## Reference: host vars (in `molecule/default/molecule.yml` and `converge.yml`)

| Override | Default | Purpose |
|---|---|---|
| `audit_git_version` | current `benchmark_<v.N.0>` branch (audit repo uses semver, not DISA-style) | Branch on `UBUNTU22-STIG-Audit` to pull goss content from. Override to test a QA branch before merging (see "Overriding the audit branch" below). |
| `audit_output_destination` | `<working_dir>/_temp_fetched_audits/` (relative to `MOLECULE_PROJECT_DIRECTORY`) | Where fetched audit JSONs land. Pinned outside the role tree so they don't pollute system paths or git. |
| `fetch_audit_output` | `true` | Auto-copies audit JSONs out of the container to `audit_output_destination`. |
| `system_is_container` | `true` | Gates container-incompatible tasks (auditd, kernel sysctls, mount changes, GRUB, AppArmor kernel-level rules, etc.) - see `vars/is_container.yml`. |
| `ubtu22stig_skip_for_test` | `true` | Bypasses the role's "you must read the disclaimer" assertion that normally guards real-host runs. |
| `ubtu22stig_subscribed` | `true` | Pretends Ubuntu Pro is active so FIPS/Pro-gated rules (e.g. `UBTU-22-211000`) don't assert-fail. |
| `ubtu22stig_set_bootloader_password` | `false` | Bootloader assertions are skipped via `system_is_container`; this is belt-and-braces. |
| `change_requires_reboot` | `false` | Suppresses reboot handlers (containers can't reboot). |
| `ansible_become` | `false` | Container is already running as root; sudo via become is unnecessary. |
| `setup_audit` / `run_audit` | `true` | Drives the pre/post audit goss runs. |

## Overriding the audit branch

If you're QA'ing a feature branch on `UBUNTU22-STIG-Audit` that hasn't been merged yet, override via extra-vars:

```bash
molecule converge -- --extra-vars 'audit_git_version=<your-qa-branch>'
```

`include_vars` has higher precedence than play/host vars in Ansible, so `--extra-vars` is the only way to change the audit branch from a Lockdown-internal `vars/audit.yml` pin.

**Branch naming reminder:** The remediation repo uses DISA-style `benchmark_v2rN` (no dots), but the audit repo uses semver `benchmark_v2.N.0`. Always pass the full audit branch name to `audit_git_version` (e.g. `benchmark_v2.7.0`, not `benchmark_v2r7`).

## Where to ask for help

- This role's open issues: `github.com/ansible-lockdown/UBUNTU22-STIG/issues`
- Ansible Lockdown Discord (linked from the role README badge)
- Molecule docs: `molecule.readthedocs.io`
- Goss output format: `github.com/goss-org/goss`
