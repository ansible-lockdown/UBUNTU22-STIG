# Ubuntu22STIG

## 2026 August - Contributing guide and README refresh

- replaced `CONTRIBUTING.rst` with `CONTRIBUTING.md`, carrying the current Ansible-Lockdown
  contributing guide. It adds the approved-contributor model for pull requests, keeps issues open to
  everyone, and retains the DCO 1.1 text and the GPG plus Signed-off-by requirements
- `README.md`: added a Contributing section pointing at `CONTRIBUTING.md`, and refreshed the
  Community Contribution section, which still described the previous open pull request workflow and
  so contradicted the new guide
- `README.md`: removed the decorative emoji from headings, and switched the social badge from
  `twitter.com` to `x.com`

## v2r8 alignment (never-imported controls, CI gates, company rename, audit source)

No benchmark version change.

- `tasks/Cat2/UBTU-22-254xxx.yml` was never imported by `tasks/Cat2/main.yml`, which went straight from `253xxx` to `255xxx`. Four CAT II controls had therefore never run on any host: UBTU-22-254010, 254015, 254020 and 254030 (SSSD package, SSSD for multifactor authentication, SSSD certificate path validation, and PKI identity mapping). The import is added in numeric order. Their CCI tags were also incomplete and now carry `CCI-004046`/`CCI-004047` (254010, 254015) and `CCI-004909` (254020)
- `ubtu22stig_254015` added to `vars/is_container.yml`. It starts the `sssd` service, and that file excludes service management inside containers. Note this means the newly activated service start is not exercised by container testing
- the audit concurrency limit is now settable: `audit_max_concurrent` is declared in `defaults/main/audit.yml` and passed to the audit runner as `-m` from both the pre- and post-remediation audit tasks. It is interpolated through `| default(50, true)`, because an empty override would otherwise consume the following `-o` argument and silently lose the scan output path
- the parent company name changed from Tyto Athene to Quantum Sky. Renamed in `meta/main.yml`, `vars/main.yml` and `LICENSE`. `vars/main.yml` feeds `file_managed_by_ansible`, so the header written into every templated file carries the new name; expect a one-off change on those files at the next run, and note that `templates/etc/modprobe.d/module.conf.j2` notifies `Change_requires_reboot` and `templates/etc/rsyslog.d/50-default.conf.j2` notifies `Restart_rsyslog`
- deliberately not renamed: the UBTU-22-411045 `blockinfile` marker in `tasks/Cat3/UBTU-22-411xxx.yml`. It is matched byte for byte against the block written by earlier releases, so changing any character would leave a duplicate faillock stanza on every host that still carries the old block. That single Company Naming finding is recorded in `.qa_baseline.json`
- deliberately not renamed: existing entries in this file, which record what was true when written
- new CI, both secret-free and running in front of the existing pipelines: `.github/workflows/molecule.yml` (container smoke test, the `default` scenario) and `.github/workflows/repo_qa.yml` (the Ansible-Lockdown QA checker pinned to `2.8.4`, run with `--strict` against `.qa_baseline.json`)
- `.gitignore` no longer ignores `.github/`. The directory was ignored while four workflow files were tracked, so any newly added workflow silently failed to stage
- `molecule/default/converge.yml`: dropped the stale V2R7 release marker from the comment header rather than bumping it, since a hardcoded release in a comment goes stale every cycle
- removed `templates/etc/aide.conf.j2` and `templates/etc/dconf/db/local.d/00-screensaver.j2`. Nothing deployed either one: the AIDE controls line-edit `/etc/aide/aide.conf` and the screensaver path is written by `community.general.ini_file`
- `tasks/Cat3/UBTU-22-411xxx.yml`: spaced the filter pipes in the faillock `insertafter`/`insertbefore` expressions
- README: removed the emoticons from the section headings and pointed the X badge image at `x.com`
- `tasks/Cat2/main.yml` imported `UBTU-22-611xxx.yml` twice under an identical task name, so every 611xxx password task ran twice. One copy removed
- `handlers/main.yml`: the ssh restart handler targeted `sshd`. The unit shipped by `openssh-server` on Ubuntu 22.04 is `ssh.service`; `sshd.service` exists only as an alias. Now `ssh`
- the audit binary source moved from the `goss-org` project at `v0.4.8` to the `krameff` fork at `v0.5.0`, with both architecture checksums updated. Prose references in `README.md` and `molecule/README_Molecule_QuickStart.md` were updated with it
- `set -o pipefail` added to all 52 shell tasks, none of which had it. A shell task whose pipe fails now fails the task instead of passing silently
- PAM is now configured through the pam-auth-update profile source instead of the generated `/etc/pam.d/common-*` files, so `UBTU-22-611060` (CAT I, no null passwords) and `UBTU-22-611055` (sha512, `rounds=100000`) survive a regeneration. New `templates/usr/share/pam-configs/pam_unix.j2`, built from the Ubuntu 22.04 stock profile with `nullok` removed and the algorithm and rounds templated; new `ubtu22stig_passwd_rounds`, `ubtu22stig_pam_confd_dir` and `ubtu22stig_pam_pwunix_file`; new `Pam_auth_update_pwunix` handler using `--enable` rather than `--force`. The `nullok` `replace`, the hardcoded rounds `lineinfile` and the role's only `community.general.pamd` call are retired. Previously the settings were removed only from the generated files while the profile source kept `nullok`, so any full regeneration reverted a CAT I control <!-- pragma: allowlist secret -->
- `molecule/default/verify.yml` now regenerates the PAM stack and asserts both controls survive it, so a future change that reintroduces the generated-file-only approach fails the suite
- the role defaults moved from a single `defaults/main.yml` to a `defaults/main/` directory holding `main.yml` and `audit.yml`, and every audit variable is now consolidated in `defaults/main/audit.yml`. `vars/audit.yml` is removed along with the `include_vars` that loaded it. This is a variable-precedence change and that is the point: those settings were previously loaded by `include_vars`, which outranks play and host vars, so only extra-vars could override them. They are now ordinary role defaults, so inventory and play vars work. Verified by setting `audit_git_version` from a play var and confirming it takes effect, which it could not before
- the Repo QA Checker pin moved to `2.8.4`, which is the first release that accepts a `defaults/main/` directory. Earlier versions look only for `defaults/main.yml` and abort before running a single check


## v2r8 alignment (code quality, no benchmark change)

Repo hygiene and pattern alignment (no rule additions, removals, or content changes):
- handlers: removed orphaned handlers never referenced by any notify/listen.
- Converted `ansible_facts.<key>` references to bracket notation `ansible_facts['<key>']`.
- Moved the audit binary/content configuration variables into `vars/audit.yml`; added `audit_bin_validate_certs` and enabled TLS validation on the audit binary download.
- Moved `warn_control_id` from block level onto the individual `warning_facts.yml` import task.
- Renamed the goss audit-vars template to `templates/lockdown_audit.yml.j2`.
- Added `.molecule/` to `.gitignore`.
- Bumped `actions/checkout` to v7 in the CI workflows.

## benchmark_v2r8 (V2R8 alignment - STIG V2R8, 01 April 2026)

V2R7 -> V2R8 is updates-only (188 rules unchanged; 0 added, 0 removed, 0 severity changes; 7 SV-* revision drifts).

V2R8 benchmark alignment:
- defaults/main.yml: benchmark_version v2.7.0 -> v2.8.0
- README.md: V2R7 reference + V2R7 download URL -> V2R8
- molecule/default/molecule.yml: comment header V2R7 -> V2R8
- molecule/default/verify.yml: benchmark_version v2.7.0 -> v2.8.0

SV-* revision-suffix updates (7 rules content-edited in V2R8):
- UBTU-22-232080: SV-260501r958566_rule -> SV-260501r1184052_rule (system-journal directory ownership root)
- UBTU-22-232085: SV-260502r958566_rule -> SV-260502r1184054_rule (system-journal directory group systemd-journal)
- UBTU-22-232090: SV-260503r958566_rule -> SV-260503r1184056_rule (system-journal file ownership root)
- UBTU-22-232095: SV-260504r958566_rule -> SV-260504r1184058_rule (system-journal file group systemd-journal)
- UBTU-22-251020: SV-260516r991593_rule -> SV-260516r1184061_rule (ufw enabled)
- UBTU-22-255020: SV-260525r958390_rule -> SV-260525r1184064_rule (DOD banner text)
- UBTU-22-271030: SV-260539r1069103_rule -> SV-260539r1184066_rule (gsettings Ctrl-Alt-Delete)

Pre-existing typo cleanups (SV-* tag references that were either missing or malformed across cycles):
- UBTU-22-213010: SV-260472r958524 -> SV-260472r1137695_rule (missing `_rule` suffix + wrong revision)
- UBTU-22-214010: SV-260476r986272 -> SV-260476r1015003_rule (missing `_rule` suffix + wrong revision)
- UBTU-22-232065: SV-260498r991560_rulee -> SV-260498r991560_rule (typo doubled `e`)

Functional updates required by V2R8 Check/Fix text:
- UBTU-22-251020: added `enabled: true` to systemd_service so the ufw service is enabled on boot (V2R8 Check verifies `Status: active`; audit goss already required `enabled: true`)
- UBTU-22-271030: ini_file value `'""'` -> `'@as []'` to match V2R8 Fix Text (typed empty array of strings) and the existing audit goss expectation

Stale reference fixes:
- README.md: Python3.8 -> Python 3.10+ requirement (ansible-core 2.16 mandates Python 3.10+ on controller)
- tasks/main.yml: sudo_password_rule sentinel UBTU-22-010380 -> UBTU-22-432010 (010380 was never in V2R7 or V2R8 XCCDF; current sudo reauthentication rule is 432010)

Molecule local-dev convenience:
- molecule/default/{molecule,converge}.yml: enabled `fetch_audit_output: true` and pinned `audit_output_destination` to `<working_dir>/_temp_fetched_audits/`, so pre/post audit JSONs auto-copy out of the container after converge for off-host inspection.
- molecule/README_Molecule_QuickStart.md: added quick-start guide for the default scenario covering prereqs, walkthrough, expected results, `PLAY RECAP` interpretation, host_vars reference, Apple Silicon note, and audit-branch override (`benchmark_v2.N.0` semver). Covers the single `default/` scenario, which is the only one this role ships.
- molecule/default/{molecule,converge}.yml: set `ubtu22stig_disruption_high: true` to exercise the full code path in the throwaway container. The role default in `defaults/main/main.yml` stays `false` for production safety. Verified no regressions; post-remediation audit failures drop 30 -> 23 by exercising journald 232080/85/90/95 + 232025 (/var/log dir mode) + 232140 (journalctl access) - controls that gate on disruption_high.

QA hardening:
- defaults/main.yml + tasks/: added `ubtu22stig_shell_executable` (default `/bin/bash`) and applied `args: executable: "{{ ubtu22stig_shell_executable }}"` to every `ansible.builtin.shell` task so piped audit/discovery shells run under bash consistently (Ubuntu's `/bin/sh` is dash, which lacks `pipefail`). Configurable for non-standard environments.
- tasks/main.yml, tasks/prelim.yml, tasks/Cat1/UBTU-22-21xxxx.yml: modernized bare injected-fact references to `ansible_facts[...]` (`virtualization_type`, `env.SUDO_USER`, `mounts`, `distribution_version`) for forward compatibility with `inject_facts_as_vars: false`.
- handlers/main.yml: removed dead `Remount_var_log` handler (not notified by any task).
- molecule/default/{molecule,converge}.yml: corrected the `audit_output_destination` relative path so fetched audit JSONs resolve to `<working_dir>/_temp_fetched_audits/` as intended.
- molecule/README_Molecule_QuickStart.md: documented the `ubtu22stig_shell_executable` default in the gotchas section.

Full-QA findings closed:
- UBTU-22-654230 + UBTU-22-412035: corrected task titles that had been copied verbatim from a neighboring control (bodies and SV-* tags were already correct); titles now match the V2R8 XCCDF.
- UBTU-22-255030 / 255035 / 255045: fixed malformed legacy V-ID tags (`SV-260527/260528/260530` -> `V-260527/260528/260530`); the canonical `SV-...r..._rule` tags were already correct.
- UBTU-22-611035 + UBTU-22-653065: added the missing `CAT2` severity tag.
- README.md: spelling/grammar fixes ("complaint" -> "compliant", "preSTIGion" -> "precision", "This have" -> "This has").
- .gitignore: added a secrets block (`*.vault`, `*.key`, `*.pem`, `*vault_pass*`, etc.) and `*qa_report*`; removed a stray pre-cycle QA report HTML from the working tree.
- UBTU-22-612040: create baseline /etc/pam_pkcs11/pam_pkcs11.conf when absent under disruption_high and drop backrefs from the use_mappers lineinfile so the STIG scanner check for V-260579 no longer fails on systems without the pam-pkcs11 package (addresses #30). Note: a custom use_mappers value is replaced with pwent.

## 2026_MAY_QA_V2R7 (QA cycle on V2R7 - no benchmark version change)

Repo hygiene and pattern fixes (no rule additions, removals, or content changes):

- Bumped min_ansible_version from 2.12.1 to 2.16.1 in meta/main.yml and vars/main.yml
- meta/main.yml: author Mark Bolwell -> Ansible-Lockdown Team; company -> MindPoint Group - A Tyto Athene Company
- LICENSE: fixed Mindpoint -> MindPoint casing
- README: V2R6 reference and download URL updated to V2R7; Twitter -> X badge rebrand; Ansible 2.12+ -> 2.16+; removed libselinux-python and python-def (RHEL leakage)
- CONTRIBUTING.rst: header "MindPoint Group Projects" -> "Ansible-Lockdown Projects"
- tasks/Cat3/UBTU-22-411xxx.yml: faillock blockinfile marker Mindpoint -> MindPoint
- tasks/main.yml: container detection now covers community.docker.docker; file mode 'u=rwx,go=rx' -> 'go-w'
- handlers/main.yml: removed orphan RHEL Firewalld_reload handler; added sshd -t pre-validate handler with listen: Restart_ssh; fixed lowercase notify change_requires_reboot to Change_requires_reboot
- tasks/{pre,post}_remediation_audit.yml and tasks/prelim.yml: 5 tasks reordered register: after changed_when/failed_when
- templates/ansible_vars_goss.yml.j2: ubtu22stig_ssh_required hardcoded true -> {{ ubtu22stig_ssh_required }}
- Removed vars/AlmaLinux.yml (RHEL template leakage, never loaded)
- Renamed Changelog.md -> CHANGELOG.md (case standardisation)

## V2R7 alignment - based on STIG v2r7 (05 Jan 2026)

Rule updates per v2r7_history.md:
- UBTU-22-211000 - Release dates to Discussion and subscription status check command
- UBTU-22-212015 - /etc/default/grub and /boot/grub/grub.cfg (audit=1 both GRUB lines)
- UBTU-22-213015 - Check output and intent updated
- UBTU-22-215040 - New: prohibit installation of NFS packages
- UBTU-22-254010 - Second finding statement in Check
- UBTU-22-254025 - Removed duplicate requirement
- UBTU-22-255050 - Removed aes192-ctr from Ciphers; no order requirement
- UBTU-22-432010 - NOPASSWD removed from check and fix text
- UBTU-22-432011 - Rule title to contain OS name (Ubuntu 22.04 LTS)
- UBTU-22-631015 - Updated finding statement
- UBTU-22-651015 - Corrected spacing in Check and Fix
- UBTU-22-654041 - Removed auditctl from Fix (role already aligned - no auditctl in template)
- Rule numbers updated throughout (CMS)
- benchmark_version set to v2.7.0

## 11th Feb 2026 - based on v2r6

- pre-commit-update
- workflow updates
- pre status update thanks to @kurtcorsha
- sudo group variable moved to correct section in defaults/main.ym


## 2nd December 2025 - based on v2r6

Rule updates
- UBTU-22-211000 - New control
- UBTU-22-212015
- UBTU-22-232026
- UBTU-22-651015

several other improvements
- ssh kex,macs and cipher logic
- 232026, 232080, 232090, 232140 fixed stig consistency errors
- 214010 - logic update
- 411045 - rewritten due to missing steps in STIG documentation for common-account

### 23 October 2025 - based on STIG v2r5
Control updates from Version 2 Release 3 through Version 2 Release 5
- New README layout
- New Workflows

Rules updates

CAT1
- UBTU-22-271030
- UBTU-22-432015 - updated logic
- UBTU-22-611060
  - Added loop

CAT2
- UBTU-22-212015
- UBTU-22-213015
- UBTU-22-232020
  - extended find locations
- UBTU-22-232070
  - extended find locations
- UBTU-22-232075
  - extended find locations
- UBTU-22-232110
- UBTU-22-253010
- UBTU-22-254010 - New control
- UBTU-22-254015 - New control
- UBTU-22-254020 - New control
- UBTU-22-254030 - New control
- UBTU-22-271020
- UBTU-22-271025
- UBTU-22-432010
- UBTU-22-432011 - New control
- UBTU-22-611055
- UBTU-22-612020
- UBTU-22-612030
- UBTU-22-651015
- UBTU-22-651030
- UBTU-22-653030
- UBTU-22-653065
- UBTU-22-653075
  - Added loop control
- UBTU-22-654041 - New control
- UBTU-22-654224 - New control

CAT3
- UBTU-22-252010
- UBTU-22-252015
- UBTU-22-254025 - New control
- UBTU-22-411045 rewritten
- UBTU-22-412015 - Removed
- UBTU-22-653020
- UBTU-22-653035 conditional fix

# Based on STIG v2r2 - final updates

- pre-commit updates
- workflow tidy up
- #21 addressed thanks to @kurtCorsha
- #23 addressed thanks to @kurtCorsha

## Based on STIG v2r2

### 1.0.0

Renaming of prefix on vars
UFW fix on controls section UBTU-22-251xxx

## Based on STIG v2r2

### 1.0.0

Updated lint configs
spacing aligned
lint updates

### Initial
