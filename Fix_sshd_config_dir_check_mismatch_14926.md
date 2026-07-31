# Contribution: Some ssh rules write remediation to /etc/ssh/sshd_config.d/ but check /etc/ssh/sshd_config

**Student:** Jonathan Morales

**Issue:** [https://github.com/ComplianceAsCode/content/issues/14926](https://github.com/ComplianceAsCode/content/issues/14926)

**Status:** Phase II, In Progress

---

## Why I Chose This Issue

I picked this issue because it has a clear, reproducible bug report with exact commands and output from the reporter. The rule cannot be removed and there is no ambiguity about what correct behavior should look like, since the remediation itself already understands distributed sshd config and the check simply does not. I also chose it over another candidate issue that was opened by a maintainer but described as possibly just a testing environment problem, since that issue's root cause was not established yet.

---

## Understanding the Issue

### Problem Description

Several sshd rules write their remediation to a drop in file under `/etc/ssh/sshd_config.d/` when the product uses a distributed sshd config, but the OVAL check for these rules only reads `/etc/ssh/sshd_config`. After remediation runs successfully, the check still fails because it never looks in the directory where the value was actually written.

### Expected Behavior

After remediation runs on a distributed config system, the check should pass, since the required value is present somewhere sshd will read it from.

### Current Behavior

The reporter showed that after remediation writes `Ciphers aes256-ctr,aes192-ctr,aes128-ctr` to `/etc/ssh/sshd_config.d/00-complianceascode-hardening.conf` on SLES 15 SP5, the check still returns an error, since the OVAL object only reads `/etc/ssh/sshd_config`.

### Affected Components

The issue names four rules: `sshd_use_approved_ciphers`, `sshd_use_approved_ciphers_ordered_stig`, `sshd_use_approved_macs`, `sshd_use_approved_macs_ordered_stig`. All four live under `linux_os/guide/services/ssh/ssh_server/`. Each has an `oval/shared.xml`, `bash/shared.sh`, and `ansible/shared.yml`.

---

## Reproduction Process

### Environment Setup

I reattached to the existing `ssg-dev` docker container rather than creating a new one. The SSH agent does not persist across container restarts, so `git fetch upstream` failed with a publickey error until I ran `eval "$(ssh-agent -s)"` and `ssh-add ~/.ssh/id_ed25519` again. The key itself was already registered on GitHub, confirmed with `ssh -T git@github.com`, so no new key was needed.

I built the sle15 datastream with `./build_product sle15 --datastream`, since sle15 is a product that sets `sshd_distributed_config`, matching the SLES 15 SP5 environment in the bug report.

### Steps to Reproduce

1. Generate the actual remediation bash script for `sshd_use_approved_ciphers` against the built sle15 datastream using `oscap xccdf generate fix`, with the stig profile selected.
2. Confirm in that generated script that the Ciphers value is written only to `/etc/ssh/sshd_config.d/00-complianceascode-hardening.conf`, never to `/etc/ssh/sshd_config` directly.
3. Extract the OVAL component from the same built datastream with `oscap ds sds-split`, and locate the `textfilecontent54_object` for the Ciphers check.
4. Confirm that object uses a single hardcoded `<ind:filepath>/etc/ssh/sshd_config</ind:filepath>`, with no reference to the config directory.
5. Build a minimal fake root at `/tmp/fakeroot`, with an empty `sshd_config` and the real Ciphers line placed only in `sshd_config.d/00-complianceascode-hardening.conf`, matching what remediation actually produces.
6. Write a minimal standalone OVAL document reproducing the same object and state as the real check, and evaluate it against the fake root with `OSCAP_PROBE_ROOT=/tmp/fakeroot oscap oval eval`.
7. Observe the result.

### Reproduction Evidence

Running the standalone OVAL document against the fake root returned:

```
Definition oval:repro-broken:def:1: false
```

This matches the reporter's actual result, since the check fails even though the correct value is present on disk, just in the location the check does not read.

My findings: the object definition confirmed directly from the built datastream matches exactly what I reproduced, a single `<ind:filepath>` with no path or filename pattern that would also match the config directory.

---

## Solution Approach

### Analysis

The check and the remediation for these four rules disagree on where the configuration lives. The remediation macro, `bash_sshd_remediation`, already accepts a `config_is_distributed` parameter and correctly writes to the config directory when set. The OVAL side was never given the same awareness for these four rules specifically.

I found that other rules in the same codebase already solve this correctly. `sshd_rekey_limit` and `sshd_use_strong_ciphers` both call a shared macro, `sshd_oval_check`, which accepts `config_is_distributed` and builds separate check logic for the main file and the config directory.

Looking at the actual matching logic used by `sshd_use_approved_ciphers` and `sshd_use_approved_macs`, I found they use a different comparison style than `sshd_use_approved_ciphers_ordered_stig` and `sshd_use_approved_macs_ordered_stig`. The plain variants split the configured value and the approved value list on commas and check that at least one approved value is present, an intersection check against a dynamic external variable. The ordered_stig variants use a single fixed regex, the same style already used successfully by `sshd_use_strong_ciphers`.

I confirmed, by reading the actual macro source in `shared/macros/10-oval.jinja`, that `sshd_oval_check` and its supporting macros only build exact equals or fixed pattern matches. None of them support the comma split and intersection logic the two plain variants need. Extending the shared macro to support that would affect every other rule that already uses it, so I posted a scoping comment on the issue asking for direction before touching that logic.

The two ordered_stig variants do not have this problem, since their fixed regex matching style is the same style `sshd_oval_check` already supports through `sshd_use_strong_ciphers`. This plan only covers those two rules for now.

### Proposed Solution

For `sshd_use_approved_ciphers_ordered_stig` and `sshd_use_approved_macs_ordered_stig`, replace the hand written OVAL body with a call to the shared `sshd_oval_check` macro, passing the existing fixed regex as the value and `config_is_distributed=sshd_distributed_config`, the same call shape already used by `sshd_use_strong_ciphers`.

### Implementation Plan

Using UMPIRE framework:

**Understand:** The OVAL check for these two rules only reads `/etc/ssh/sshd_config`, while their remediation can write to `/etc/ssh/sshd_config.d/` on distributed config products, so the check fails after remediation succeeds.

**Match:** `sshd_use_strong_ciphers` already solves this exact shape of problem with a fixed regex value, calling `sshd_oval_check` with `config_is_distributed=sshd_distributed_config`.

**Plan:**
1. Replace the hand written `oval/shared.xml` body in both rules with a call to `sshd_oval_check`, using the existing fixed regex and `config_is_distributed=sshd_distributed_config`.
2. Confirm `bash/shared.sh` and `ansible/shared.yml` for both rules already pass `config_is_distributed=sshd_distributed_config` to their remediation macros. Already confirmed for the ciphers ordered_stig variant. Still need to confirm the macs ordered_stig variant directly.
3. Rebuild the sle15 datastream and run the same fake root reproduction method against the real generated OVAL for both rules, to confirm the result changes from false to true.
4. Check whether existing tests in each rule's `tests/` folder already cover the config directory scenario, since `sshd_use_approved_macs_ordered_stig` already has a test file named `correct_value_config_dir.pass.sh` that appears to expect this scenario to pass.

**Implement:** Not started. No branch created yet for this issue.

**Review:** Confirm no other product changes behavior unexpectedly, since `sshd_oval_check` is shared by other already passing rules. Confirm `rule.yml` field order stays unchanged in both rules.

**Evaluate:** Rebuild the datastream, extract the OVAL, and run the same `OSCAP_PROBE_ROOT` fake root test method used in reproduction, confirming the result flips from false to true for both rules.

---

## Testing Strategy

### Unit Tests

- [ ] Confirm `sshd_use_approved_macs_ordered_stig/tests/correct_value_config_dir.pass.sh` passes once the fix is applied
- [ ] Confirm existing main file test scenarios for both rules still pass after the fix

### Integration Tests

- [ ] Rebuild sle15 datastream and re-run the fake root reproduction against the real generated OVAL for both rules, confirming the result is true

### Manual Testing

Not yet performed. No code has been written for this issue.

---

## Implementation Notes

### Week 1 Progress

Confirmed the root cause directly from the built sle15 datastream rather than assuming it from the issue text. Reproduced the exact failure using a fake root and a standalone OVAL document evaluated with `oscap oval eval`. Found two real working precedents already in the codebase, `sshd_rekey_limit` and `sshd_use_strong_ciphers`, both using the shared `sshd_oval_check` macro. Read the actual macro source for `sshd_oval_check`, `oval_line_in_file_object`, and `oval_line_in_file_state` to confirm none of them support the comma split and intersection logic used by `sshd_use_approved_ciphers` and `sshd_use_approved_macs`. Posted a scoping comment on the issue asking for direction on those two rules before touching shared macro code. No branch created yet, no fix written yet for the two ordered_stig rules covered by this plan.

### Code Changes

Not started.

---

## Pull Request

Not opened yet.

---

## Learnings & Reflections

### Technical Skills Gained

Learned how to isolate a single OVAL object and state pair into a standalone document and evaluate it against a fake filesystem root using `OSCAP_PROBE_ROOT`, without needing a live target system or a working Automatus backend. This let me prove the bug and the fix pattern with real tool output instead of only reading source code.

### Challenges Overcome

An early grep for `oval-def:definition` style tags returned nothing for `sshd_use_strong_ciphers`, which looked like the rule might not exist in the sle15 build at all. Checking the actual tag prefix used in the built datastream showed the grep pattern itself was wrong, not the rule's presence. This was a useful reminder to verify a negative result before treating it as a real finding.

### What I'd Do Differently Next Time

I would check the actual tag naming in a built datastream before writing a grep pattern based on assumption, rather than after getting an empty result that looked like a real finding.

---

## Resources Used

- [ComplianceAsCode/content issue #14926](https://github.com/ComplianceAsCode/content/issues/14926)
- [ComplianceAsCode Style Guide](https://complianceascode.readthedocs.io/en/latest/manual/developer/04_style_guide.html)
- [ComplianceAsCode Contributing Guidelines](https://github.com/ComplianceAsCode/content/blob/master/CONTRIBUTING.md)
