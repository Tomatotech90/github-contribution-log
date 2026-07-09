# Contribution #3: Remove DoD Specific Verbiage from rule.yml Files (Part 2) update

**Contribution Number:** 3

**Student:** Jonathan Morales

**Issue:** [https://github.com/ComplianceAsCode/content/issues/8709](https://github.com/ComplianceAsCode/content/issues/8709)

**Prior PR:** [https://github.com/ComplianceAsCode/content/pull/14834](https://github.com/ComplianceAsCode/content/pull/14834)

**This PR:** [https://github.com/ComplianceAsCode/content/pull/14841](https://github.com/ComplianceAsCode/content/pull/14841)

**Status:** Phase IV, In Progress, Open, first round of maintainer feedback received and addressed (commit `5f56e413f1`), awaiting further review

---

## Update: Maintainer Feedback and Revert

Two changes were requested after review.

First, 8 of the 15 files are STIG-specific or mirror upstream STIG content, so the DoD wording should remain rather than be reworded. These were reverted to the original text:

- `ssh_use_approved_macs_ordered_stig/rule.yml`
- `set_firewalld_default_zone/policy/stig/shared.yml`
- `harden_sshd_macs_opensshserver_conf_crypto_policy/policy/stig/shared.yml`
- `set_password_hashing_algorithm_systemauth/policy/stig/shared.yml`
- `set_password_hashing_algorithm_systemauth/policy/stig/rhel10.yml`
- `accounts_password_pam_minlen/policy/stig/shared.yml`
- `sssd_certificate_verification/policy/stig/shared.yml`
- `var_smartcard_drivers.var`

Each file's original text was pulled directly from git history rather than reconstructed and confirmed with a diff against upstream after reverting.

Second, 4 banner files needed a stronger edit. Rewording the lead-in sentence ("The DoD required text is either:") was not enough, as the actual DoD banner text remained. For 3 of these, the required-text block was removed entirely from the description field:

- `banner_etc_issue_net/rule.yml`
- `banner_etc_motd/rule.yml`
- `banner_etc_gdm_banner/rule.yml`

The 4th, `banner_etc_profiled_ssh_confirm/rule.yml`, was left unchanged. Its required text is embedded in a shell script's `read -p` prompt rather than a standalone block, so removing it isn't as simple as deleting a paragraph. A question was posted to the maintainer asking whether to remove the whole script or just the wording inside it, and this file is on hold until that's answered.

All changes were verified with grep, the project's CI-equivalent yamllint check, and a full build (`./build_product rhel9 --datastream`), all of which matched the original submission.

## Update: var_ssh_confirm_text Generic Option

Follow-up feedback suggested using the existing `var_ssh_confirm_text` variable instead of hardcoded text in `banner_etc_profiled_ssh_confirm/rule.yml`.

The variable already existed and was already used correctly in the fix script and the OVAL check. It only had one option, `dod_default`, with no generic alternative.

Added a second option, `generic_default`, to `var_ssh_confirm_text.var`. Generated the regex for the new banner text with the project's own tool, `utils/regexify_banner.py`, the same tool used for `dod_default`. Updated `rule.yml`'s description field to explain that the text is configurable through the variable instead of showing the DoD text directly.

Confirmed the new option resolves correctly by running it through the same regexify steps the build uses. The result was clean, valid banner text. Also confirmed with yamllint and a full product build.

An existing STIG control file explicitly pins `var_ssh_confirm_text` to `dod_default`, so this change does not affect that profile.

## Testing

<!-- FEEDBACK: Phase III - Testing -->

No new automated test file was added for this contribution. All 15 files in scope and the 11 files touched in this revision are prose-only edits to `description`, `rationale`, `vuldiscussion`, or similar text fields. None of the changes touch OVAL checks, remediation scripts (bash/ansible), or rule logic, so no existing behavior being tested actually changed.

Existing tests for all affected rules were not modified and are expected to pass as is, since none of them assert on the edited prose fields. This was confirmed indirectly through the full product build below, which would fail if any structural or logic issue were introduced.

**What was tested and how:**

1. **grep verification**: confirmed target DoD text was removed (or restored, for the 8 reverted files) in each affected file. See "Before State Confirmation" and the revert verification in the Update section above.
2. **Lint verification**: ran the project's CI-equivalent yamllint check (piping Jinja-templated files through `utils/strip_jinja_for_yamllint.py`) against all 11 changed files. Only pre-existing warnings were found; no new errors. One pre-existing error (`var_smartcard_drivers.var`, trailing spaces) was confirmed to already exist in `upstream/master` and is unrelated to this contribution.
3. **Build verification**: ran `./build_product rhel9 --datastream` after all edits, completed cleanly (`[13/13] Updating data stream ssg-rhel9-ds.xml to 1.3`).
4. **Diff verification against upstream**: for the 8 reverted files, ran a direct `diff` against `upstream/master` (or the file's actual parent commit, where master had since drifted) and confirmed a byte-for-byte match with zero remaining differences.

No manual testing beyond the above was performed, since these are documentation-only fields with no runtime behavior to exercise manually.

## Acceptance Criteria

<!-- FEEDBACK: Phase IV - Acceptance criteria checklist -->

- [x] Follows the project's style guide for prose edits (no field reordering, no logic changes)
- [x] No breaking changes to OVAL checks, remediations, or rule structure
- [x] Existing tests unaffected (see Testing section above)
- [ ] New automated test added: not applicable, see Testing section for rationale
- [x] All maintainer feedback addressed: `banner_etc_profiled_ssh_confirm` now uses the `var_ssh_confirm_text` generic option, see Update section above

## Original Submission


---

## Why This Contribution

Contribution #2 (PR #14834) addressed 5 files from the grep list for issue #8709 as a first pass. After that PR was merged, the remaining 32 files were reviewed in full to determine which ones could be safely generalized and which ones needed maintainer input first. This PR addresses 15 of those 32 files. The remaining 17 are documented at the end of this README with an explanation of why each group needs clarification before being touched.

---

## Understanding the Issue

### Problem Description

Issue #8709 asks contributors to remove DoD-specific phrasing from `rule.yml` files across the project so that content describes security requirements in policy-agnostic language. The issue has been open since May 2022, and the maintainer posted a grep list of 119 remaining occurrences across 37 files on October 9, 2025.

The primary fix mechanism is either removing the DoD-specific sentence when it adds no technical value, replacing it with a generic equivalent language, or substituting a hardcoded value with an XCCDF variable reference when one already exists. The five files in contribution #2 used the removal-and-rewording approach. This PR uses the same approach for 15 more files.

### Files In Scope for This PR

```
linux_os/guide/services/ssh/ssh_client/ssh_use_approved_macs_ordered_stig/rule.yml
linux_os/guide/system/network/network-firewalld/ruleset_modifications/set_firewalld_default_zone/policy/stig/shared.yml
linux_os/guide/system/software/integrity/crypto/harden_sshd_macs_opensshserver_conf_crypto_policy/policy/stig/shared.yml
linux_os/guide/system/software/integrity/crypto/configure_gnutls_tls_crypto_policy/rule.yml
linux_os/guide/system/accounts/accounts-physical/screen_locking/smart_card_login/var_smartcard_drivers.var
linux_os/guide/system/accounts/accounts-pam/set_password_hashing_algorithm/set_password_hashing_algorithm_systemauth/policy/stig/shared.yml
linux_os/guide/system/accounts/accounts-pam/set_password_hashing_algorithm/set_password_hashing_algorithm_systemauth/policy/stig/rhel10.yml
linux_os/guide/system/accounts/accounts-pam/password_quality/password_quality_pwquality/accounts_password_pam_minlen/policy/stig/shared.yml
linux_os/guide/services/sssd/sssd_certificate_verification/policy/stig/shared.yml
linux_os/guide/services/http/securing_httpd/httpd_public_resources_not_shared/rule.yml
linux_os/guide/system/software/updating/ensure_gpgcheck_repo_metadata/rule.yml
linux_os/guide/system/accounts/accounts-banners/banner_etc_profiled_ssh_confirm/rule.yml
linux_os/guide/system/accounts/accounts-banners/banner_etc_motd/rule.yml
linux_os/guide/system/accounts/accounts-banners/gui_login_banner/banner_etc_gdm_banner/rule.yml
linux_os/guide/system/accounts/accounts-banners/banner_etc_issue_net/rule.yml
```

---

## Reproduction Process

### Environment Setup

Repository cloned locally from `git@github.com:Tomatotech90/content.git` on Ubuntu 24.04 VM. Upstream remote set to `git@github.com:ComplianceAsCode/content.git`. Working branch: `fix-dod-verbiage-rule-yml-part2`.

### Steps Taken to Identify Affected Files

1. After PR #14834 was merged, the remaining files from the maintainer's October 9, 2025 grep list were located:

```bash
find . -type f \( \
  -path "*/ssh_use_approved_macs_ordered_stig/rule.yml" -o \
  -path "*/sssd_certificate_verification/policy/stig/shared.yml" -o \
  -path "*/set_firewalld_default_zone/policy/stig/shared.yml" -o \
  -path "*/ensure_gpgcheck_repo_metadata/rule.yml" -o \
  -path "*/harden_sshd_macs_opensshserver_conf_crypto_policy/policy/stig/shared.yml" -o \
  -path "*/configure_gnutls_tls_crypto_policy/rule.yml" -o \
  -path "*/var_smartcard_drivers.var" -o \
  -path "*/smartcard_auth/rule.yml" -o \
  -path "*/set_password_hashing_algorithm_systemauth/policy/stig/shared.yml" \
\)
```

2. Each file was read in full (not just the grep snippet) to understand the full context around each DoD mention before deciding how to handle it.

3. Each occurrence was placed into one of three categories: incidental prose that can be safely reworded (this PR), hardcoded values that need an XCCDF variable check (deferred), or DoD-specific content that is the entire substance of the rule (excluded, see Notes section).

### Before State Confirmation

Before any edits were applied, the following grep confirmed the DoD-specific text was present across all 15 files:

```
linux_os/guide/services/ssh/ssh_client/ssh_use_approved_macs_ordered_stig/rule.yml
    DoD Information Systems are required to use FIPS-approved cryptographic hash
    functions. The only hash algorithms meeting this requirement is SHA2.

linux_os/guide/system/network/network-firewalld/ruleset_modifications/set_firewalld_default_zone/policy/stig/shared.yml
    It also permits outbound connections that may facilitate exfiltration of DoD data.

linux_os/guide/system/software/integrity/crypto/harden_sshd_macs_opensshserver_conf_crypto_policy/policy/stig/shared.yml
    Remote access (e.g., RDP) is access to DoD nonpublic information systems by an
    authorized user...

linux_os/guide/system/software/integrity/crypto/configure_gnutls_tls_crypto_policy/rule.yml
    title: 'Configure GnuTLS library to use DoD-approved TLS Encryption'
    To verify if GnuTLS uses defined DoD-approved TLS Crypto Policy, run:
    Configure the ... GnuTLS library to use only DoD-approved encryption...
    must implement DoD-approved TLS encryption in the GnuTLS package.

linux_os/guide/system/accounts/accounts-physical/screen_locking/smart_card_login/var_smartcard_drivers.var
    For DoD, choose the cac driver.

linux_os/guide/system/accounts/accounts-pam/set_password_hashing_algorithm/.../shared.yml
    and DoD data may be compromised.
    ...meets DoD requirements.

linux_os/guide/system/accounts/accounts-pam/set_password_hashing_algorithm/.../rhel10.yml
    and DoD data may be compromised.
    ...meets DoD requirements.

linux_os/guide/system/accounts/accounts-pam/password_quality/.../accounts_password_pam_minlen/.../shared.yml
    The DoD minimum password requirement is 15 characters.

linux_os/guide/services/sssd/sssd_certificate_verification/policy/stig/shared.yml
    Using an authentication device, such as a DoD Common Access Card (CAC)...
    ...and the DoD CAC.

linux_os/guide/services/http/securing_httpd/httpd_public_resources_not_shared/rule.yml
    In addition to the requirements of the DoD Internet-NIPRNet DMZ STIG...

linux_os/guide/system/software/updating/ensure_gpgcheck_repo_metadata/rule.yml
    NOTE: For U.S. Military systems, this requirement does not mandate DoD certificates...

linux_os/guide/system/accounts/accounts-banners/banner_etc_profiled_ssh_confirm/rule.yml
    The DoD required text is:

linux_os/guide/system/accounts/accounts-banners/banner_etc_motd/rule.yml
    The DoD required text is either:

linux_os/guide/system/accounts/accounts-banners/gui_login_banner/banner_etc_gdm_banner/rule.yml
    The DoD required text is either:

linux_os/guide/system/accounts/accounts-banners/banner_etc_issue_net/rule.yml
    The DoD required text is either:
```

---

## Solution Approach

### Analysis

Each of the 15 files in this PR contains DoD-specific phrasing in a prose field (`rationale`, `vuldiscussion`, `description`, `title`, `fixtext`, `ocil`) that applies to any organization. None of them contains a hardcoded value that should be expressed as an XCCDF variable. The correct fix for all 15 is the removal or rewording of the DoD-specific text, the same approach the maintainer confirmed in PR #13622 and PR #14834.

### Fix Approach

All edits were applied in a single Python script using exact string matching to avoid any risk of regex-related whitespace or indentation errors. For each file, the script confirmed that the target text existed before writing the change and clearly reported a no match if none was found.

The full before and after for each file:

**ssh_use_approved_macs_ordered_stig/rule.yml** (rationale)
```
Before:
    DoD Information Systems are required to use FIPS-approved cryptographic hash
    functions. The only hash algorithms meeting this requirement is SHA2.

After:
    FIPS-approved cryptographic hash functions are required for protecting the
    integrity of communications. The only hash algorithms meeting this requirement
    are SHA2-based algorithms.
```

**set_firewalld_default_zone/policy/stig/shared.yml** (vuldiscussion)
```
Before:
    It also permits outbound connections that may facilitate exfiltration of DoD data.

After:
    It also permits outbound connections that may facilitate exfiltration of sensitive data.
```

**harden_sshd_macs_opensshserver_conf_crypto_policy/policy/stig/shared.yml** (vuldiscussion)
```
Before:
    Remote access (e.g., RDP) is access to DoD nonpublic information systems by an
    authorized user (or an information system) communicating through an external,
    non-organization-controlled network.

After:
    Remote access (e.g., RDP) is access to nonpublic information systems by an
    authorized user (or an information system) communicating through an external,
    non-organization-controlled network.
```

**configure_gnutls_tls_crypto_policy/rule.yml** (title, ocil, fixtext, srg_requirement)
```
Before:
    title: 'Configure GnuTLS library to use DoD-approved TLS Encryption'
    To verify if GnuTLS uses defined DoD-approved TLS Crypto Policy, run:
    ...use only DoD-approved encryption by adding...
    must implement DoD-approved TLS encryption in the GnuTLS package.

After:
    title: 'Configure GnuTLS library to use Approved TLS Encryption'
    To verify if GnuTLS uses the defined approved TLS Crypto Policy, run:
    ...use only approved encryption by adding...
    must implement approved TLS encryption in the GnuTLS package.
```

**var_smartcard_drivers.var** (description)
```
Before:
    <br />For DoD, choose the cac driver.

After:
    (line removed entirely)
```

**set_password_hashing_algorithm_systemauth/policy/stig/shared.yml** (vuldiscussion, two changes)
```
Before:
    and DoD data may be compromised.
    ...meets DoD requirements.

After:
    and sensitive data may be compromised.
    ...meets organizational requirements.
```

**set_password_hashing_algorithm_systemauth/policy/stig/rhel10.yml** (vuldiscussion, same two changes)
```
Before:
    and DoD data may be compromised.
    ...meets DoD requirements.

After:
    and sensitive data may be compromised.
    ...meets organizational requirements.
```

**accounts_password_pam_minlen/policy/stig/shared.yml** (vuldiscussion)
```
Before:
    The DoD minimum password requirement is 15 characters.

After:
    (sentence removed entirely; the checktext and fixtext already enforce
    15 characters without DoD framing)
```

**sssd_certificate_verification/policy/stig/shared.yml** (vuldiscussion, two changes)
```
Before:
    Using an authentication device, such as a DoD Common Access Card (CAC) or token
    that is separate from the information system...
    ...and smart cards such as the U.S. Government Personal Identity Verification
    (PIV) card and the DoD CAC.

After:
    Using an authentication device, such as a hardware token or smart card that is
    separate from the information system...
    ...and smart cards such as the U.S. Government Personal Identity Verification
    (PIV) card.
```

**httpd_public_resources_not_shared/rule.yml** (rationale)
```
Before:
    In addition to the requirements of the DoD Internet-NIPRNet DMZ STIG that
    isolates inbound traffic from external network to the internal network,

After:
    In addition to the requirements of applicable DMZ segmentation policies that
    isolate inbound traffic from the external network to the internal network,
```

**ensure_gpgcheck_repo_metadata/rule.yml** (rationale)
```
Before:
    NOTE: For U.S. Military systems, this requirement does not mandate DoD
    certificates for this purpose; however, the certificate used to verify
    the software must be from an approved Certificate Authority.

After:
    NOTE: For regulated systems, this requirement does not mandate
    organization-specific certificates for this purpose; however, the
    certificate used to verify the software must be from an approved
    Certificate Authority.
```

**banner_etc_profiled_ssh_confirm/rule.yml** (description)
```
Before:
    The DoD required text is:

After:
    The required text is:
```

**banner_etc_motd/rule.yml** (description)
```
Before:
    The DoD required text is either:

After:
    The required text is either:
```

**banner_etc_gdm_banner/rule.yml** (description)
```
Before:
    The DoD required text is either:

After:
    The required text is either:
```

**banner_etc_issue_net/rule.yml** (description)
```
Before:
    The DoD required text is either:

After:
    The required text is either:
```

---

## Testing

### Verify No DoD References Remain

After all edits were applied, the following grep returned no output across all 15 files:

```bash
grep -rn "DoD" \
  linux_os/guide/services/ssh/ssh_client/ssh_use_approved_macs_ordered_stig/rule.yml \
  linux_os/guide/system/network/network-firewalld/ruleset_modifications/set_firewalld_default_zone/policy/stig/shared.yml \
  linux_os/guide/system/software/integrity/crypto/harden_sshd_macs_opensshserver_conf_crypto_policy/policy/stig/shared.yml \
  linux_os/guide/system/software/integrity/crypto/configure_gnutls_tls_crypto_policy/rule.yml \
  linux_os/guide/system/accounts/accounts-physical/screen_locking/smart_card_login/var_smartcard_drivers.var \
  linux_os/guide/system/accounts/accounts-pam/set_password_hashing_algorithm/set_password_hashing_algorithm_systemauth/policy/stig/shared.yml \
  linux_os/guide/system/accounts/accounts-pam/set_password_hashing_algorithm/set_password_hashing_algorithm_systemauth/policy/stig/rhel10.yml \
  linux_os/guide/system/accounts/accounts-pam/password_quality/password_quality_pwquality/accounts_password_pam_minlen/policy/stig/shared.yml \
  linux_os/guide/services/sssd/sssd_certificate_verification/policy/stig/shared.yml \
  linux_os/guide/services/http/securing_httpd/httpd_public_resources_not_shared/rule.yml \
  linux_os/guide/system/software/updating/ensure_gpgcheck_repo_metadata/rule.yml \
  linux_os/guide/system/accounts/accounts-banners/banner_etc_profiled_ssh_confirm/rule.yml \
  linux_os/guide/system/accounts/accounts-banners/banner_etc_motd/rule.yml \
  linux_os/guide/system/accounts/accounts-banners/gui_login_banner/banner_etc_gdm_banner/rule.yml \
  linux_os/guide/system/accounts/accounts-banners/banner_etc_issue_net/rule.yml

(no output, all DoD references removed)
```

### CI-Equivalent Lint Check

The project's `ci_lint.yml` workflow pipes Jinja-templated files through `utils/strip_jinja_for_yamllint.py` before linting. Running the same logic locally:

```
=== ssh_use_approved_macs_ordered_stig/rule.yml ===
stdin
  4:1  warning  too many blank lines (3 > 2)  (empty-lines)

=== set_firewalld_default_zone/policy/stig/shared.yml ===
(clean)

=== harden_sshd_macs_opensshserver_conf_crypto_policy/policy/stig/shared.yml ===
(clean)

=== configure_gnutls_tls_crypto_policy/rule.yml ===
(clean)

=== var_smartcard_drivers.var ===
(clean)

=== set_password_hashing_algorithm_systemauth/policy/stig/shared.yml ===
(clean)

=== set_password_hashing_algorithm_systemauth/policy/stig/rhel10.yml ===
stdin
  1:1  warning  too many blank lines (1 > 0)  (empty-lines)

=== accounts_password_pam_minlen/policy/stig/shared.yml ===
stdin
  17:1  warning  too many blank lines (15 > 2)  (empty-lines)

=== sssd_certificate_verification/policy/stig/shared.yml ===
(clean)

=== httpd_public_resources_not_shared/rule.yml ===
(clean)

=== ensure_gpgcheck_repo_metadata/rule.yml ===
(clean)

=== banner_etc_profiled_ssh_confirm/rule.yml ===
stdin
  57:1  warning  too many blank lines (1 > 0)  (empty-lines)

=== banner_etc_motd/rule.yml ===
(clean)

=== banner_etc_gdm_banner/rule.yml ===
(clean)

=== banner_etc_issue_net/rule.yml ===
(clean)
```

All warnings are pre-existing (present in the files before any edit was made) and do not cause CI to fail. The project's `ci_lint.yml` only fails on errors (exit code 1), not warnings (exit code 2).

Pre-existing trailing whitespace was also found in `var_smartcard_drivers.var` and `banner_etc_profiled_ssh_confirm/rule.yml` during the lint run. Both were fixed in the same commit since those files were already being touched.

### Build Check

```bash
./build_product rhel9 --datastream
```

```
-- RHEL 9: ON
-- Scanning for dependencies of rhel9 fixes...
-- Configuring done (8.4s)
-- Generating done (0.0s)
-- Build files have been written to: .../content/build
[13/13] [rhel9-content] Updating data stream ssg-rhel9-ds.xml to 1.3
```

Build completed cleanly with all 15 edited files in place.

---

## Implementation Notes

**Commit:** `df6dcb9397`

**Branch:** `fix-dod-verbiage-rule-yml-part2`

**Approach decisions:** All 15 edits were applied in a single Python exact-string-match script rather than `sed`, since several edits span multiple lines or contain characters that would need escaping in a regex context. Running the script twice on already-edited files produces no changes (idempotent). Each file was read in full before editing rather than relying on the few lines of context shown in the maintainer's grep output.

---

## Pull Request

**PR Link:** [https://github.com/ComplianceAsCode/content/pull/14841](https://github.com/ComplianceAsCode/content/pull/14841)

**Contribution Log:** [https://github.com/Tomatotech90/github-contribution-log/blob/main/Remove_DoD_Specific_Verbiage_from_rule.yml_(Part%202).md](https://github.com/Tomatotech90/github-contribution-log/blob/main/Remove_DoD_Specific_Verbiage_from_rule.yml_(Part%202).md)

<!-- FEEDBACK: Phase IV - Maintainer Feedback log -->

**Maintainer Feedback:**

- First round of feedback: revert 8 STIG-specific/upstream-mirrored files back to original DoD wording, since these files mirror upstream STIG content. For 4 banner files, remove the required banner text entirely from the `description` field rather than just rewording the lead-in sentence.
- Response (commit `5f56e413f1`): reverted the 8 files to their original upstream text, confirmed with a direct diff against `upstream/master` for each. Removed the required-text block entirely from 3 of the 4 banner files (`banner_etc_issue_net`, `banner_etc_motd`, `banner_etc_gdm_banner`). Left the 4th (`banner_etc_profiled_ssh_confirm/rule.yml`) unchanged, since its DoD text is embedded inside a shell script's `read -p` prompt rather than a standalone block. Posted a follow-up question asking whether to remove the script or only the wording.
- Second round of feedback: use an XCCDF variable (`var_ssh_confirm_text`) in place of the hardcoded DoD text in `banner_etc_profiled_ssh_confirm/rule.yml`.
- Investigated and confirmed `var_ssh_confirm_text.var` already exists and is already wired into the fix script (`bash/shared.sh`) and the OVAL check (`oval/shared.xml`) via `external_variable`. The variable had only one option (`dod_default`) with no generic alternative. Confirmation received that adding another option would be fine.
- Response: added a new option, `generic_default`, to `var_ssh_confirm_text.var`, generated with the project's own `utils/regexify_banner.py` tool. Updated `rule.yml`'s description field to reference the variable instead of showing the DoD text directly. Confirmed the new option resolves correctly through the same deregexify steps used at build time, and confirmed with yamllint and a full product build. See the Update section above for details.

**Status:** Open; first round of feedback addressed; second round (`var_ssh_confirm_text` generic option) implemented and verified ---- awaiting maintainer review.

---

## Notes: 17 Files Excluded from This PR

After reviewing all 32 remaining files from the maintainer's grep list, 17 were excluded from this PR because the DoD reference in each one is not incidental wording around a generic requirement. Instead, it is either the verbatim text that a compliance check verifies against, a named DoD infrastructure or network requirement, or a DoD-specific PKI requirement with no clear generic equivalent. Editing these without maintainer guidance would risk changing what the rule actually checks for or misrepresenting its purpose.

These 17 files fall into three groups. Questions for each group are included in the PR description.

### Group 1: Banner Text Files (7 files)

```
linux_os/guide/system/accounts/accounts-banners/banner_etc_issue/rule.yml
linux_os/guide/system/accounts/accounts-banners/banner_etc_issue/policy/stig/shared.yml
linux_os/guide/system/accounts/accounts-banners/gui_login_banner/gui_login_dod_acknowledgement/rule.yml
linux_os/guide/system/accounts/accounts-banners/login_banner_text.var
linux_os/guide/system/accounts/accounts-banners/motd_banner_text.var
linux_os/guide/system/accounts/accounts-banners/remote_login_banner_text.var
linux_os/guide/services/http/securing_httpd/httpd_secure_content/var_web_login_banner_text.var
```

These files contain the Standard Mandatory DoD Notice and Consent Banner as verbatim text. The `.var` files use option keys named `dod_banners` and `dod_default,`, which DoD STIG profiles select by name. The `rule.yml` files reference those option keys in their checks. Renaming the keys or removing the banner text would break profile selections and change what is actually verified. Maintainer guidance is needed before touching any of these.

### Group 2: PKI and Certificate Files (5 files)

```
linux_os/guide/services/sssd/sssd_has_trust_anchor/rule.yml
linux_os/guide/services/sssd/sssd_has_trust_anchor/policy/stig/shared.yml
linux_os/guide/services/http/securing_httpd/httpd_modules_improve_security/httpd_deploy_mod_ssl/httpd_configure_valid_server_cert/rule.yml
linux_os/guide/system/network/network_ssl/only_allow_dod_certs/rule.yml
linux_os/guide/services/http/securing_httpd/httpd_secure_content/httpd_configure_banner_page/rule.yml
```

These rules check for DoD-specific PKI infrastructure. `sssd_has_trust_anchor` verifies the presence of `CN = DoD Root CA 3` in the certificate store and points to `cyber.mil` as the source for the root CA file. `only_allow_dod_certs` exists specifically to enforce DoD PKI. `httpd_configure_valid_server_cert` checks for `OU = DoD` in the certificate issuer fields. These rules appear to be intentionally DoD-specific. Maintainer guidance is needed on whether a generic equivalent exists or whether these rules should remain as-is.

### Group 3: DoD Infrastructure and Network Files (5 files)

```
linux_os/guide/services/http/securing_httpd/httpd_nipr_accredited_dmz/rule.yml
linux_os/guide/system/accounts/accounts-physical/screen_locking/smart_card_login/smartcard_auth/rule.yml
linux_os/guide/services/ntp/chronyd_server_directive/rule.yml
linux_os/guide/services/ntp/chronyd_or_ntpd_set_maxpoll/rule.yml
linux_os/guide/system/software/integrity/endpoint_security_software/mcafee_security_software/group.yml
```

`httpd_nipr_accredited_dmz` is a rule about DoD NIPRNet infrastructure; the entire title and requirement are DoD-specific. `smartcard_auth` contains a DoD-specific exemption list as part of the rule's substance. `chronyd_server_directive` and `chronyd_or_ntpd_set_maxpoll` have NIPRNet/SIPRNet terminology in their `srg_requirement` fields, which appear to be direct verbatim quotes from the STIG SRG. `mcafee_security_software` was already excluded from PR #14834 for the same reason: the rule exists specifically to mandate McAfee HBSS in DoD environments. Maintainer guidance is needed on whether SRG verbatim quotes should be left as-is and whether the other files in this group have generic equivalents.

---

## Learnings

### Technical Skills Gained

Reading files in full before triaging them was essential here. Several files that appeared easy based on the two-line grep snippet turned out to contain DoD references that were load-bearing once the full file context was visible. The banner `.var` files, in particular, look like simple string replacements in the grep output. Still, they actually define option keys that STIG profiles reference by name, so changing the key names would silently break profile selections.

<!-- FEEDBACK: Phase IV - Challenges Overcome -->

### Challenges Overcome

Reverting 8 files accurately required not guessing at the original wording. The contribution log only had a written summary of the "before" text, not confirmed formatting or whitespace, so reconstructing it by hand risked introducing a subtly wrong revert. Pulling the actual original text directly from git history (`git show upstream/master:<path>`) and diffing the result against the same source afterward solved this, since it removed any need to trust a written summary over the actual repository history. One file (`set_password_hashing_algorithm_systemauth/policy/stig/rhel10.yml`) initially produced a diff full of unrelated content; tracing this back showed `upstream/master` had moved ahead of the PR's branch point for unrelated reasons (a separate yescrypt/sha512 rewrite), so the correct comparison had to be made against that file's actual parent commit instead of the current master.

The `banner_etc_profiled_ssh_confirm/rule.yml` file initially appeared to need a new variable created from scratch, based on the maintainer's suggestion to use an XCCDF variable. Checking the actual fix script, the OVAL check, and the existing `.var` file directly showed that the variable already existed and was correctly wired in. The actual gap was no a  generic option.

### What Would Be Done Differently

The 17 hard files should have questions posted on the issue thread when the PR is opened, so the maintainer can address both in a single review pass rather than sequentially.

---

## Resources Used

- [ComplianceAsCode/content issue #8709](https://github.com/ComplianceAsCode/content/issues/8709)
- [ComplianceAsCode/content PR #14834 (prior PR, merged)](https://github.com/ComplianceAsCode/content/pull/14834)
- [ComplianceAsCode/content PR #14841 (this PR)](https://github.com/ComplianceAsCode/content/pull/14841)
- [ComplianceAsCode/content PR #13622 (prior contributor, merged)](https://github.com/ComplianceAsCode/content/pull/13622)
- [ComplianceAsCode Style Guide](https://complianceascode.readthedocs.io/en/latest/manual/developer/04_style_guide.html)
- [ComplianceAsCode Contributing Guidelines](https://github.com/ComplianceAsCode/content/blob/master/CONTRIBUTING.md)
