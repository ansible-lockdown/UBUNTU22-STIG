# Ubuntu22STIG

## Hotfix - UBTU-22-612040 pam_pkcs11.conf baseline (addresses #30)

- UBTU-22-612040: create a baseline /etc/pam_pkcs11/pam_pkcs11.conf when absent under disruption_high, and drop backrefs from the use_mappers lineinfile so the STIG scanner check for V-260579 no longer fails on systems without the pam-pkcs11 package (addresses #30). Note: a custom use_mappers value is replaced with pwent.

## V2R7 - QA cycle (no benchmark version change)

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
- #23 addressed thanks to @kurtCorshsa

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
