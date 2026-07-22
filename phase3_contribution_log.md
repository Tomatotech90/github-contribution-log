# Contribution #3: Remove DoD Specific Verbiage from rule.yml Files (Part 2)

**Contribution Number:** 3
**Student:** Jonathan Morales
**Issue:** [https://github.com/ComplianceAsCode/content/issues/8709](https://github.com/ComplianceAsCode/content/issues/8709)
**Status:** Phase III Complete

---

## Implementation Notes

### Week 1 Progress

Built the initial 15-file submission. Went through the remaining files from the maintainer's grep list on issue #8709 (119 occurrences across 37 files) and triaged each one into three categories: incidental prose that could be safely generalized, hardcoded values that would need an XCCDF variable, and DoD-specific content that is the actual substance of the rule.

15 files were edited in this pass:
- `ssh_use_approved_macs_ordered_stig/rule.yml`
- `set_firewalld_default_zone/policy/stig/shared.yml`
- `harden_sshd_macs_opensshserver_conf_crypto_policy/policy/stig/shared.yml`
- `configure_gnutls_tls_crypto_policy/rule.yml`
- `var_smartcard_drivers.var`
- `set_password_hashing_algorithm_systemauth/policy/stig/shared.yml`
- `set_password_hashing_algorithm_systemauth/policy/stig/rhel10.yml`
- `accounts_password_pam_minlen/policy/stig/shared.yml`
- `sssd_certificate_verification/policy/stig/shared.yml`
- `httpd_public_resources_not_shared/rule.yml`
- `ensure_gpgcheck_repo_metadata/rule.yml`
- `banner_etc_profiled_ssh_confirm/rule.yml`
- `banner_etc_motd/rule.yml`
- `banner_etc_gdm_banner/rule.yml`
- `banner_etc_issue_net/rule.yml`

Commit: `df6dcb9397` — Remove DoD-specific verbiage from rule.yml files (part 2)

17 additional files from the grep list were reviewed in full and excluded, since the DoD reference in each is the actual substance of the rule (verbatim banner text a check verifies against, a specific PKI certificate authority, a direct SRG quote), not incidental wording. These were documented in three groups in the PR description with open questions for the maintainer, and remain open as of this submission.

### Week 2 Progress

Received maintainer feedback (Mab879) requesting two changes: revert 8 STIG-specific files back to original DoD wording since they mirror upstream STIG content, and remove the required banner text entirely (not just reword the lead-in sentence) from 4 banner files.

Reverted the 8 files using `git show upstream/master:<path>` to pull the actual original text from git history rather than reconstructing it by hand, then confirmed each file matched upstream exactly with a direct `diff`. One file, `set_password_hashing_algorithm_systemauth/policy/stig/rhel10.yml`, produced a diff full of unrelated content on the first attempt; tracing it back showed `upstream/master` had moved ahead of this branch's fork point for unrelated reasons (a separate yescrypt/sha512 rewrite), so the correct comparison had to be made against that file's actual parent commit (`6398afee5c`) instead of current master.

Removed the required-text block entirely from 3 of the 4 banner files (`banner_etc_issue_net`, `banner_etc_motd`, `banner_etc_gdm_banner`). The 4th, `banner_etc_profiled_ssh_confirm/rule.yml`, was left unchanged in this commit, since its required text sits inside a shell script's `read -p` prompt rather than a standalone block, and I did not want to guess whether to remove the whole script or just the wording without maintainer input.

Commit: `5f56e413f1` — Address review feedback on DoD verbiage removal (part 2) (11 files changed, 13 insertions, 93 deletions)

### Week 3 Progress

Maintainer suggested using an existing XCCDF variable, `var_ssh_confirm_text`, in place of the hardcoded DoD text in `banner_etc_profiled_ssh_confirm/rule.yml`. Investigated the variable directly (`var_ssh_confirm_text.var`, `bash/shared.sh`, `oval/shared.xml`) and found it already existed and was already wired into the fix script and OVAL check via `external_variable`, but only had one option, `dod_default`, with no generic alternative.

Added a second option, `generic_default`, to `var_ssh_confirm_text.var`, generating its regex with the project's own tool (`utils/regexify_banner.py`), the same tool used to generate `dod_default`. Updated `rule.yml`'s description to reference the variable instead of showing DoD text directly.

Commit: `5e7e1a4c7f` — Add generic option to var_ssh_confirm_text for banner_etc_profiled_ssh_confirm

Attempted to add an automated test scenario (`banner_etc_profiled_ssh_confirm_generic.pass.sh`) for the new option, following the exact pattern of the existing `dod_default` test. This went through several iterations:

- Commit `6c3831823b` — added the test scenario file
- Commit `4498786d45` — added `# variables = var_ssh_confirm_text=generic_default` metadata so the scan would select the new option
- Commit `299096d454` — fixed a wording mismatch between the test's banner text and the actual stored regex
- Commit `0d48f7f3fc` — escaped an apostrophe in the banner text that was breaking Jinja template parsing in CI (confirmed via the actual CI error: `TemplateSyntaxError: expected token 'end of statement block', got 's'`)

After the apostrophe fix resolved the Jinja parsing error, CI still failed the same test with "Rule evaluation resulted in fail, instead of expected pass." Traced this through the actual test framework source (`tests/ssg_test_suite/rule.py`, specifically `generate_xslt_change_value_template`) and confirmed the `variables` metadata mechanism performs a blind, literal text substitution into the datastream's unselected `<value>` node. It does not resolve a named option like `generic_default` into its actual regex content, and the regex itself cannot be placed directly in the metadata line either, since `variables` is parsed as a comma-separated list and the regex contains many literal commas that would be split apart during parsing. This is a genuine limitation of the test framework for regex-backed, multi-option variables, not a mistake in the test file itself.

Reported this finding to the maintainer with the specific code reference. He confirmed and offered to test the option manually instead.

Commit `1568131f4f` — removed the test scenario per that direction.

### Week 4 Progress

Maintainer did manual testing and found two further issues with the `generic_default` regex itself:

1. An unescaped apostrophe in the banner text was causing remediation to error out at runtime (separate from the earlier Jinja parsing issue, this was in the actual remediation path). Reworded the banner text to drop the possessive phrasing entirely rather than re-escaping it, removing the class of bug instead of patching around it (commit `c03651a812`).
2. A core bug in the shared `escape_regex` function in `ssg/utils.py`: an unescaped `?` was being interpreted as a regex quantifier instead of a literal character during remediation. The maintainer supplied a tested patch; I applied it exactly as given and regenerated the `generic_default` regex with the corrected tool so it also picked up the fix (commit `ede308b220`).

Maintainer also flagged one remaining nit: the `srg_requirement` field in `configure_gnutls_tls_crypto_policy/rule.yml` should have stayed verbatim (STIG SRG fields mirror the SRG text exactly), since it wasn't incidental wording like the rest of that file's edits. Reverted just that one field back to the original DoD wording, confirmed against upstream with a direct diff, leaving the title/ocil/fixtext fields generalized as before.

Commit: *[hash pending confirmation — final push output was not captured in this session; needs to be added before submission]*

### Code Changes

- **Files modified across this contribution:** 16 files total (see full list above), plus one shared utility file, `ssg/utils.py`
- **Key commits:** `df6dcb9397`, `5f56e413f1`, `5e7e1a4c7f`, `6c3831823b`, `4498786d45`, `299096d454`, `0d48f7f3fc`, `1568131f4f`, `c03651a812`, `ede308b220`
- **Approach decisions:** Used `git show upstream/master:<path>` rather than reconstructing original text by hand for every revert, to guarantee byte-for-byte accuracy. Used the project's own `utils/regexify_banner.py` tool to generate regex values rather than hand-writing them, and cross-validated the tool's output by regenerating the known `dod_default` regex from source text and diffing it against the stored value. Traced test framework behavior through actual Python source rather than accepting AI-generated (Copilot) suggestions at face value; two of three Copilot suggestions during this contribution were incorrect when checked against the actual codebase (one proposed changing this project's double-brace Jinja delimiter convention, which is real and working elsewhere in the codebase; another proposed remediation logic for a different rule mechanism than the one this rule actually uses).

---

## Testing Strategy

### Unit Tests

- [ ] A new automated test scenario for `generic_default` was attempted (`banner_etc_profiled_ssh_confirm_generic.pass.sh`) but ultimately removed, since the test framework's `variables` metadata mechanism cannot resolve a named option into a regex-backed value (confirmed by reading `tests/ssg_test_suite/rule.py`'s `generate_xslt_change_value_template`, which performs a literal text substitution, not an option lookup). The maintainer confirmed this and tested the option manually instead.
- [x] Existing tests for all other 15 modified files were prose-only edits (`description`, `rationale`, `vuldiscussion`, `srg_requirement` fields) with no changes to OVAL checks, remediation logic, or rule structure, so no existing test scenario needed modification.

### Integration Tests

- [x] Full product build (`./build_product rhel9 --datastream` and `./build_product ubuntu2404 --datastream-only`) run after every change in this contribution, confirming the datastream compiles cleanly end to end.
- [x] Manual round-trip verification: reproduced the exact `sed` transformation pipeline used by the project's `bash_deregexify_banner_*` Jinja macros (`shared/macros/10-bash.jinja`) against the stored `generic_default` regex, confirming it resolves to clean, valid, correctly-formatted shell script text before the maintainer's own manual test.

### Manual Testing

Verified all 8 reverted files matched upstream `master` exactly using direct `diff` comparisons (`diff <(git show upstream/master:<path>) <path>`), confirming zero remaining differences on each. Verified the 3 banner-text removals produced clean, correctly-structured YAML with `git diff` review of each hunk. Ran the project's CI-equivalent yamllint check (piping Jinja-templated files through `utils/strip_jinja_for_yamllint.py`, since plain yamllint misparses this project's `{{{ }}}` templating syntax) on every changed file across the whole contribution; only pre-existing warning-level issues were found, no new errors introduced.

---

## Challenges Faced

**Reverting files accurately without a live reference.** The safest approach was to treat git history as the single source of truth rather than reconstructing "before" text from documentation, since a hand-typed reconstruction risks a subtle whitespace or wording error. One file's parent commit had to be identified specifically (rather than using current `master`) because `master` had drifted ahead of this branch on that file for unrelated reasons.

**A test framework limitation that looked like a bug in my own file at first.** After three separate rounds of fixes to the test scenario (metadata syntax, wording, apostrophe escaping) still failed CI, the responsible move was to stop guessing and read the actual Python source that applies test scenario metadata. That confirmed the `variables` mechanism cannot express a regex-backed option selection at all, a real limitation of the framework, not a mistake in the test file. Reporting this precisely, with the exact function and file reference, let the maintainer make an informed call quickly rather than continuing an unproductive cycle.

**Distinguishing AI-suggested fixes from verified ones.** During CI troubleshooting, an AI coding assistant (Copilot) proposed several fixes that sounded plausible but were checked against the actual project source before being applied. Two suggestions were found to be wrong once verified: one claimed this project's `{{% %}}` Jinja delimiter syntax was invalid, disproven by the fact that the pre-existing, already-passing `dod_default` test uses the identical syntax; another proposed remediation code for `sshd_config`/`Banner` directives that has nothing to do with how this specific rule actually works (`/etc/profile.d/ssh_confirm.sh`). Neither suggestion was applied without first confirming it against the real codebase.

---

## Resources Used

- [ComplianceAsCode/content issue #8709](https://github.com/ComplianceAsCode/content/issues/8709)
- [ComplianceAsCode/content PR #14841 (this contribution)](https://github.com/ComplianceAsCode/content/pull/14841)
- [ComplianceAsCode Contributing Guidelines](https://github.com/ComplianceAsCode/content/blob/master/CONTRIBUTING.md)
- Project's own `utils/regexify_banner.py` and `tests/ssg_test_suite/rule.py` source, read directly to confirm mechanisms rather than relying on documentation or AI-generated summaries alone
