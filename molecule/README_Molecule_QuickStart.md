# Molecule Quick Start - Ubuntu 22.04 LTS STIG

> Audience: anyone new to molecule testing with this Ansible Lockdown role. Read top-to-bottom the first time; come back to the reference tables later.

## What this scenario does

This `molecule/` directory ships one scenario: `default/`. It spins up a throwaway Ubuntu 22.04 LTS Docker container, applies the **whole** Ansible Lockdown Ubuntu 22.04 STIG role against it, then runs the paired audit with syver (content from the `UBUNTU22-STIG-Audit` repo) to see how many controls passed and how many still fail. It's the gating test you run before merging or pushing benchmark changes.

End-to-end you get:

1. A clean container booted into systemd.
2. Pre-remediation audit (baseline): records which controls are already failing before the role runs.
3. Role converge #1: applies all the STIG remediations.
4. Idempotence: `molecule idempotence` re-runs the role and fails on any changed task.
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
| Local clones of **both** this role AND `UBUNTU22-STIG-Audit` (or network access so the audit repo can be cloned at runtime) | The role pulls audit content from the audit repo during converge | `git -C <path-to>/UBUNTU22-STIG-Audit status` |

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
cd <path-to-this-role>

# Full gating pair
molecule destroy && molecule create && molecule converge && molecule idempotence && molecule verify
```

The whole sequence takes 5-15 minutes on a modern laptop (smaller package set and lighter container than the RHEL family). Network is needed (clone of audit content + apt installs) for the first run; subsequent runs reuse the cached Docker image.

## What each command does

| Command | What happens |
|---|---|
| `molecule destroy` | Removes any leftover container from a previous run. Safe to run on a clean system. |
| `molecule converge` (first) | (a) pulls / starts the `geerlingguy/docker-ubuntu2204-ansible:latest` container, (b) runs `prepare.yml` to install supporting packages, (c) clones the audit content from `UBUNTU22-STIG-Audit`, (d) runs the pre-remediation audit, (e) applies the full role, (f) runs the post-remediation audit, (g) fetches both audit JSONs to your local `_temp_fetched_audits/` folder. |
| `molecule idempotence` | Re-runs the role against the already-remediated container, skipping tasks tagged `molecule-idempotence-notest` (the audit scans, the audit fetch, the facts template and the UBTU-22-254015 `sssd` service start). Any other changed task fails the step. |
| `molecule verify` | Runs the assertions in `verify.yml`, a narrow regression check on top of converge: the post scan has no more failures than the pre scan, and after `pam-auth-update --package --force` the PAM settings of UBTU-22-611060, 611055, 411045 and 412010 are still present. The broad picture is in the audit JSONs. |

## Where the audit results land

After `molecule converge` completes, look at:

```
<working-tree>/_temp_fetched_audits/
  ubuntu2204-UBUNTU22-STIG-v<ver>_pre_scan_<timestamp>.json
  ubuntu2204-UBUNTU22-STIG-v<ver>_post_scan_<timestamp>.json
```

The JSONs are syver output - each has a `results: []` array where every entry has a `successful: true|false` field. Compare pre vs. post:

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
ubuntu2204 : ok=178  changed=67  unreachable=0  failed=0  skipped=234  rescued=0  ignored=0
```

| Counter | What it means |
|---|---|
| `ok` | Tasks that ran and finished successfully (no state change needed, or already in desired state) |
| `changed` | Tasks that modified the system (installed a package, edited a file, etc.) |
| `failed` | Tasks that errored out. **MUST BE 0** for the run to be considered passing. |
| `skipped` | Tasks gated off by `when:` conditions (often because `system_is_container: true` skips container-incompatible work) |
| `rescued` / `ignored` | Block-level error handling - typically 0 |

**Idempotence expectation:** `molecule idempotence` must show `changed=0`. The audit scans, the audit fetch and the facts template always re-run by design, and the UBTU-22-254015 `sssd` service start reports changed on every container run (ansible-lockdown/UBUNTU22-STIG#40), so those carry the `molecule-idempotence-notest` tag and are skipped by that step. Anything else that reports changed is a task that is not idempotent and needs fixing.

## Why some audit failures are expected (even after the role passes)

The post-scan will still show some controls failing. Most fall into three buckets that aren't role bugs - they're inherent to containerized testing:

1. **SSH-config controls** (banner, ciphers, KexAlgorithms, MACs, idle timeouts, login grace time): on a fresh Ubuntu container `sshd` isn't running (we install `openssh-server` but don't start the daemon - containers use `docker exec` for shell access). The audit checks for active `/etc/ssh/sshd_config` enforcement; on real hosts these controls pass after a `Restart sshd` handler fires.

2. **User-state controls** (password complexity history, password lifetime, dotfile audit): need real interactive users (UID >= 1000) with shadow entries and home directories. Containers only have system accounts, so the controls can't satisfy their preconditions.

3. **Kernel / mount / boot controls**: any control gated by `not system_is_container` is intentionally skipped during remediation (auditd kernel rules, AppArmor profiles backed by kernel modules, kernel sysctls, mount options, GRUB / boot CMDLINE) because containers share the host kernel. Audit still runs those tests, so they report as failing.

None of these warrant fixing the role. They're the expected delta between a containerized test and a real Ubuntu 22.04 host.

## Common gotchas

| Symptom | Likely cause | Fix |
|---|---|---|
| `molecule converge` errors with "container not running" | Stale container from a previous interrupted run | Run `molecule destroy` first, then retry |
| Pre-task fails with "Unable to acquire the dpkg frontend lock" | Another apt process is running (cron, unattended-upgrades) inside the prepared container | Re-run; if persistent, add a 30s sleep at the top of `prepare.yml` |
| `/bin/sh: 1: set: Illegal option -o pipefail`, or a piped shell behaving oddly (Ubuntu `/bin/sh` is dash) | A `shell:` task running under dash instead of bash | The role forces bash on every `shell:` task via `args: executable: "{{ ubtu22stig_shell_executable }}"` (default `/bin/bash`, set in `defaults/main.yml`), so this should not occur. If you add a new shell task, include that same `args:` block; grep `tasks/` for any `ansible.builtin.shell` without it. |
| Converge #2 fails on a task that passed in #1 | Ansible 2.19+ struct-vs-string type-check tripping a `when:` clause that's "lucky" on the first run | Inspect the offending task's `when:` - look for quoted-string-as-boolean bugs |
| `Conditional result (True) was derived from value of type 'str'` | Ansible 2.19+ rejects when-clauses whose final value is a non-boolean string. Common bug: an `or "<some expression>"` where the RHS got wrapped in quotes by accident | Remove the quotes; the RHS should be a bare Jinja expression |
| `molecule idempotence` fails on a task | That task changes something on every run | Make the task idempotent; tag it `molecule-idempotence-notest` only if it reports state by design, as the audit tasks do |

## Apple Silicon (M-series Mac) note

`geerlingguy/docker-ubuntu2204-ansible:latest` is a multi-arch image (amd64 + arm64). `molecule.yml` sets `platform: "${MOLECULE_DOCKER_PLATFORM:-linux/arm64}"`, so on Apple Silicon it runs the arm64 variant natively - no Rosetta required.

CI exports `MOLECULE_DOCKER_PLATFORM=linux/amd64`, because the CI runners are amd64. Export the same to reproduce it locally; on Apple Silicon that path requires Docker Desktop's "Use Rosetta for x86_64/amd64 emulation" toggle and runs noticeably slower. On an amd64 Linux host, export it to run natively.

## Reference: what's in the default scenario

| File | Purpose |
|---|---|
| `molecule.yml` | Driver: docker. Image: `geerlingguy/docker-ubuntu2204-ansible:latest` (multi-arch; native arm64 on Apple Silicon, amd64 elsewhere). systemd as PID 1 via `command: /lib/systemd/systemd`. Host vars listed in the next table. |
| `prepare.yml` | Installs supporting packages (`openssh-server`, `libpam-pwquality`, `libpam-modules`, `sudo`, `acl`, `kmod`, `cron`, `chrony`, `rsyslog`, `aide`, `aide-common`, `logrotate`, `apparmor`, `apparmor-utils`, `ufw`, `nftables`, `auditd`, `autofs`, `python3-apt`). Creates `/run/sshd`, `/etc/sysctl.d`, `/etc/ssh/sshd_config.d`. Starts `cron` and `rsyslog`. |
| `converge.yml` | Sets root password (so PAM tasks don't lock us out), stubs `/etc/default/grub` so GRUB-tag tasks don't fail in the container, includes the role. |

## Reference: scenario variables (`converge.yml` play vars, plus `ansible_become` in `molecule.yml`)

| Override | Value | Purpose |
|---|---|---|
| `audit_git_version` | current `benchmark_<v.N.0>` branch (audit repo uses semver, not DISA-style) | Branch on `UBUNTU22-STIG-Audit` to pull audit content from. Override to test a QA branch before merging (see "Overriding the audit branch" below). |
| `audit_output_destination` | `<working_dir>/_temp_fetched_audits/` (relative to `MOLECULE_PROJECT_DIRECTORY`) | Where fetched audit JSONs land. Pinned outside the role tree so they don't pollute system paths or git. |
| `fetch_audit_output` | `true` | Auto-copies audit JSONs out of the container to `audit_output_destination`. |
| `setup_audit` / `run_audit` | `true` | Drives the pre/post audit runs. |
| `ubtu22stig_disruption_high` | `true` | Applies the remediations that the role gates as disruptive, so the scan exercises them. |
| `ansible_become` | `false` | Container is already running as root; sudo via become is unnecessary. |

No other override is needed. `tasks/main.yml` detects the container and loads `vars/is_container.yml`, which turns off the container-incompatible rules. `skip_reboot` and `ubtu22stig_set_bootloader_password` already default to `true` and `false`. `ubtu22stig_subscribed` is discovered from `pro status` and cannot be overridden from play vars, because `vars/main.yml` takes precedence.

## Overriding the audit branch

If you're QA'ing a feature branch on `UBUNTU22-STIG-Audit` that hasn't been merged yet, override via extra-vars:

```bash
molecule converge -- --extra-vars 'audit_git_version=<your-qa-branch>'
```

`audit_git_version` is an ordinary role default in `defaults/main/audit.yml`, so inventory, group and play vars override it too - `--extra-vars` is simply the most direct option from the molecule command line.

**Branch naming reminder:** The remediation repo uses DISA-style `benchmark_v2rN` (no dots), but the audit repo uses semver `benchmark_v2.N.0`. Always pass the full audit branch name to `audit_git_version` (e.g. `benchmark_v2.8.0`, not `benchmark_v2r8`).

## Where to ask for help

- This role's open issues: `github.com/ansible-lockdown/UBUNTU22-STIG/issues`
- Ansible Lockdown Discord (linked from the role README badge)
- Molecule docs: `molecule.readthedocs.io`
- Audit binary (syver): `github.com/krameff/syver`
