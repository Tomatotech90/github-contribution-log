# Contribution #2: Remove DoD Specific Verbiage from rule.yml Files

**Contribution Number:** 2

**Student:** Jonathan Morales

**Issue:** [https://github.com/ComplianceAsCode/content/issues/8709](https://github.com/ComplianceAsCode/content/issues/8709)

**Status:** Phase III, In Progress

---

## Why I Chose This Issue

I have been exploring the ComplianceAsCode project, and I am genuinely impressed by the breadth of platforms and products it covers, from Red Hat Enterprise Linux and Fedora to Ubuntu, Debian, and SUSE Linux Enterprise Server, as well as products like Firefox. The goal of making it as easy as possible to write and maintain security content in all commonly used formats is something I find both practical and valuable, and the documentation, blog, and online workshops make it very approachable for new contributors. I would like to contribute to issue #8709.

---

## Understanding the Issue

### Problem Description

Several rules and group YAML files describe security requirements using organization-specific phrasing tied to DoD, even though the underlying requirement applies to any organization using the benchmark. This includes generic concepts framed as a DoD-specific mandate (audit event capability descriptions), named DoD forms standing in for a generic process (System Authorization Access Request), and named DoD-only tooling and contact references baked into otherwise generic guidance (McAfee HBSS, cyber.mil).

### Expected Behavior

Rule content should describe security requirements in policy-agnostic language. Where a concrete value genuinely varies by policy, it should be expressed through an XCCDF variable resolved per profile, not as hardcoded prose. Where the underlying requirement applies universally, DoD-specific naming should simply be removed or generalized.

### Current Behavior

Across `linux_os/guide`, dozens of `rule.yml`/`group.yml` files mix policy-agnostic security guidance with DoD-specific wording in the same paragraph, making the content read as DoD-only even when the rule itself (audit coverage of privileged commands, antivirus scanning of uploaded content, host-based intrusion detection) is generic.

### Affected Components (this PR)

- `linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml`
- `linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setfiles/rule.yml`
- `linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setsebool/rule.yml`
- `linux_os/guide/services/http/securing_httpd/httpd_configure_os_protect_web_server/httpd_antivirus_scan_uploads/rule.yml`
- `linux_os/guide/system/software/integrity/endpoint_security_software/install_hids/rule.yml`

---

## Reproduction Process

### Environment Setup

Repository cloned locally (`ComplianceAsCode/content`, `master` branch) inside the existing Ubuntu 24.04 VM used for contribution #1.

### Steps Taken to Identify Affected Files

1. Used the maintainer's October 9, 2025, grep output on the issue thread as the starting worklist (119 occurrences across 37 files).
2. Located the six candidate files locally:
   ```bash
   find . -iname "audit_rules_execution_semanage" -type d
   find . -iname "audit_rules_execution_setfiles" -type d
   find . -iname "audit_rules_execution_setsebool" -type d
   find . -iname "httpd_antivirus_scan_uploads" -type d
   find . -iname "install_hids" -type d
   find . -iname "mcafee_security_software" -type d
   ```
   This confirmed the maintainer's grep paths matched the local clone exactly.
3. Read the full content of each file rather than relying on the few lines of context shown in the issue's grep output, to see the complete surrounding `rationale`/`vuldiscussion`/`warnings` block before editing.
4. Triaged each occurrence into three categories based on whether the DoD reference was incidental prose, a value that should become an XCCDF variable, or a DoD-only requirement with no generic equivalent.
5. Confirmed the exact file and line number of each DoD reference in the five in-scope files:
   ```bash
   grep -n "DoD" \
     ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml \
     ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setfiles/rule.yml \
     ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setsebool/rule.yml \
     ./linux_os/guide/services/http/securing_httpd/httpd_configure_os_protect_web_server/httpd_antivirus_scan_uploads/rule.yml \
     ./linux_os/guide/system/software/integrity/endpoint_security_software/install_hids/rule.yml
   ```

### Reproduction Evidence

**Bug Reproduction Screenshot:**

![DoD references located via grep](bug_dod.JPG)

The screenshot above shows the exact file and line number for each DoD-specific occurrence in the five in-scope files, confirming the issue is present in the local clone before any edits are made.

**Findings:** Of the six files initially considered, five contain DoD-specific wording that is incidental to a generic security requirement and can be safely generalized. The sixth, `mcafee_security_software/group.yml`, names a DoD-mandated product (McAfee HBSS/VSEL) with no generic equivalent. This file is being set aside rather than edited (see Learnings below).

---

## Solution Approach

### Analysis

The issue's primary mechanism is replacing hardcoded policy-specific values with their corresponding XCCDF variable identifiers. For example, replacing the literal `15` in "The DoD requirement is 15" with `{{{ xccdf_value("var_accounts_password_minlen_login_defs") }}}`. That pattern applies to files where a DoD-specific numeric or string value is hardcoded in prose but already has an XCCDF variable defined for it.

The five files in this PR do not follow that pattern. None of them contains a hardcoded value that should be expressed as a variable; they contain organizational prose (descriptions of audit event categories, a named government form, a named product, and a contact URL) that happens to reference DoD as an organization. For these files, the correct fix per `ggbecker`'s guidance is deletion or rewording to make the language policy-agnostic rather than variable substitution. This is consistent with `ggbecker's October 9, 2025, confirmation that `yungcero's approach ("definitely the intended changes") covered exactly this kind of prose generalization.

### Proposed Solution

Reword each occurrence to describe the underlying security requirement without DoD-specific framing, while preserving the technical content and intent of the original text. No XCCDF variable changes are needed for this batch, since none of the affected text resolves to a profile-specific value; it's prose describing rationale, not a configurable setting.

### Fix Approach (How It Was Solved)

All three edits are combined into a single block below. It is safe to run multiple times (idempotent): the `sed`/Python steps only change text that still matches the original DoD wording, so re-running against already-fixed files is a no-op rather than a double-edit.

```bash
cd ~/Desktop/content

# Step 1: audit trio wording fix
sed -i 's/DoD has defined the list of events for which the operating system will provide an audit record generation capability as the following:/The list of events for which the operating system must provide an audit record generation capability includes the following:/' \
  ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml \
  ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setfiles/rule.yml \
  ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setsebool/rule.yml

# Step 2: httpd + install_hids wording fix (multi-line, exact-match Python replace)
python3 << 'PYEOF'
import pathlib

edits = [
    {
        "path": "linux_os/guide/services/http/securing_httpd/httpd_configure_os_protect_web_server/httpd_antivirus_scan_uploads/rule.yml",
        "old": """    Remote web authors should not be able to upload files to the Document Root
    directory structure without virus checking and checking for malicious or mobile
    code. A remote web user, whose agency has a Memorandum of Agreement (MOA) with
    the hosting agency and has submitted a DoD form 2875 (System Authorization
    Access Request (SAAR)) or an equivalent document, will be allowed to post files
    to a temporary location on the server. All posted files to this temporary
    location will be scanned for viruses and content checked for malicious or mobile
    code. Only files free of viruses and malicious or mobile code will be posted to
    the appropriate DocumentRoot directory.""",
        "new": """    Remote web authors should not be able to upload files to the Document Root
    directory structure without virus checking and checking for malicious or mobile
    code. A remote web user, whose agency has a Memorandum of Agreement (MOA) with
    the hosting agency and has submitted a System Authorization Access Request
    (SAAR) or an equivalent document, will be allowed to post files to a temporary
    location on the server. All posted files to this temporary location will be
    scanned for viruses and content checked for malicious or mobile code. Only
    files free of viruses and malicious or mobile code will be posted to the
    appropriate DocumentRoot directory.""",
    },
    {
        "path": "linux_os/guide/system/software/integrity/endpoint_security_software/install_hids/rule.yml",
        "old": """        In DoD environments, supplemental intrusion detection and antivirus tools,
        such as the McAfee Host-based Security System, are available to integrate with
        existing infrastructure. Per DISA guidance, when these supplemental tools interfere
        with proper functioning of SELinux, SELinux takes precedence. Should further
        clarification be required, DISA contact information is published publicly at
        https://www.cyber.mil/stigs/""",
        "new": """        In some environments, supplemental intrusion detection and antivirus tools
        are deployed to integrate with existing infrastructure. When these
        supplemental tools interfere with proper functioning of SELinux, SELinux
        should take precedence. Consult your organization's security guidance for
        further clarification.""",
    },
]

for edit in edits:
    p = pathlib.Path(edit["path"])
    text = p.read_text()
    if edit["old"] not in text:
        print(f"SKIPPED (already applied or text not found): {edit['path']}")
        continue
    p.write_text(text.replace(edit["old"], edit["new"]))
    print(f"Updated {edit['path']}")
PYEOF

# Step 3: trailing-whitespace cleanup on semanage (pre-existing issue, unrelated to DoD wording)
sed -i '30s/[ \t]*$//' linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml
```

**Fix application screenshot:** `sed` (Step 1) and the Python script (Step 2), with the "Updated ..." confirmation lines:

![sed and python edit commands with Updated confirmations](dod_3.JPG)

**Fix application screenshot:** the trailing-whitespace `sed` (Step 3), followed by a lint re-check:

![trailing-whitespace sed command and lint re-check](dod_1.JPG)

### Verification Order (must be run in this sequence, after the fix block above)

The fix must be applied **before** any of the checks below are meaningful. A lint or build pass run against files that still contain the original DoD wording does not verify this fix; it only verifies the file is syntactically valid, which was already true before any edit.

1. Confirm wording is actually gone (see "No DoD references remain" in Testing Strategy below)
2. Run the project's CI-equivalent lint check (see Testing Strategy)
3. Run the full product build (see Testing Strategy)

Each of the three checkboxes in Testing Strategy should be marked complete only after it has been run, **after** step 1 above confirms the wording fix is present in the working tree.

### Pre-Existing Lint Issue Discovered (Not Caused by This PR)

Running plain `yamllint` against the three audit rule files initially produced a `syntax error: could not find expected ':'`. Investigation traced this to a top-level Jinja macro call (`{{{ ocil_fix_srg_privileged_command(...) }}}`) that plain YAML misreads as the start of a flow-mapping. This was confirmed to be **pre-existing and unrelated to this PR** by parsing the original, unedited file directly: the same error occurs at the same line before any DoD-wording edit is applied.

The project's own CI (`.github/workflows/ci_lint.yml`) already accounts for this: any file containing Jinja constructs is piped through `utils/strip_jinja_for_yamllint.py` before linting, rather than being linted directly. Reproducing that exact CI logic locally:

```bash
for file in \
  linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml \
  linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setfiles/rule.yml \
  linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setsebool/rule.yml \
  linux_os/guide/services/http/securing_httpd/httpd_configure_os_protect_web_server/httpd_antivirus_scan_uploads/rule.yml \
  linux_os/guide/system/software/integrity/endpoint_security_software/install_hids/rule.yml
do
  echo "=== $file ==="
  if grep -qP '\{\{[%{#]' "$file"; then
    python3 utils/strip_jinja_for_yamllint.py "$file" | yamllint -c .yamllint -
  else
    yamllint -c .yamllint "$file"
  fi
done
```

confirmed all five files lint clean under the project's actual CI logic, aside from the one trailing-whitespace error fixed in Step 3 above and one pre-existing `empty-lines` warning in `install_hids/rule.yml` (warning only, does not fail CI, left untouched as out of scope).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Several `rule.yml` files contain DoD-specific phrasing wrapped around otherwise generic security rationale, which the issue asks to be made policy-agnostic.

**Match:** A prior PR (#13622) already established the precedent that DoD-specific text should be generalized or removed when it doesn't carry policy-specific meaning; this PR follows that same pattern for additional files surfaced by the maintainer's later grep.

**Plan:**
1. Update the `vuldiscussion` boilerplate sentence in the three audit rule files (`audit_rules_execution_semanage`, `audit_rules_execution_setfiles`, `audit_rules_execution_setsebool`) to remove the DoD-specific framing while keeping the numbered list of audited events unchanged.
2. Update the `rationale` block in `httpd_antivirus_scan_uploads/rule.yml` to drop the named DoD form reference while keeping the SAAR process description generic.
3. Update the `warnings.general` block in `install_hids/rule.yml` to remove the named product (McAfee) and the DoD-only contact URL, generalizing to "supplemental intrusion detection and antivirus tools" and "your organization's security guidance."
4. Leave `mcafee_security_software/group.yml` untouched pending maintainer input, since its DoD-specific content is the entire substance of the rule, not incidental wording.
5. Run `yamllint` / `yamlfix` against all five modified files.
6. Build the affected product datastream locally (`./build_product rhel9 --datastream`) to confirm nothing breaks.

**Implement:** [Link to branch/commits as work progresses]

**Review:** Confirm field order in each modified `rule.yml` is unchanged (no fields added, removed, or reordered; edits are prose-only within existing `vuldiscussion`/`rationale`/`warnings` blocks), and confirm none of the edited text shifts the underlying technical meaning of the requirement.

**Evaluate:** Re-read each modified file end-to-end after editing to confirm the surrounding paragraph still reads naturally, and the security intent is unchanged.

---

## Testing Strategy

### Unit Tests

No new Automatus test scenarios are required, since this PR makes no changes to OVAL checks, remediations, or rule logic, only prose wording. Existing tests for the five affected rules should pass unmodified.

### Integration Tests

**Verification Status Note:** An earlier lint and build pass was recorded for this repo, but a subsequent re-clone (to reconfirm the original bug state) reset all file edits, including the trio's wording fix. The lint/build passes that were recorded did not actually prove the wording fix, since neither checks prose content; they only validate YAML syntax and build mechanics, which pass regardless of whether the DoD wording is present. All three boxes below are reset to unchecked until reconfirmed against the actual edited content in one pass.

- [x] No DoD references remain in the five modified files
  ```bash
  grep -n "DoD" \
    ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml \
    ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setfiles/rule.yml \
    ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setsebool/rule.yml \
    ./linux_os/guide/services/http/securing_httpd/httpd_configure_os_protect_web_server/httpd_antivirus_scan_uploads/rule.yml \
    ./linux_os/guide/system/software/integrity/endpoint_security_software/install_hids/rule.yml
  ```
  (expected: no output, confirming all DoD references were removed)
  ![after fix grep confirmation](dod_4.JPG)

- [x] Project's CI-equivalent lint check passes on all five modified files
  ```bash
  for file in \
    linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml \
    linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setfiles/rule.yml \
    linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setsebool/rule.yml \
    linux_os/guide/services/http/securing_httpd/httpd_configure_os_protect_web_server/httpd_antivirus_scan_uploads/rule.yml \
    linux_os/guide/system/software/integrity/endpoint_security_software/install_hids/rule.yml
  do
    echo "=== $file ==="
    if grep -qP '\{\{[%{#]' "$file"; then
      python3 utils/strip_jinja_for_yamllint.py "$file" | yamllint -c .yamllint -
    else
      yamllint -c .yamllint "$file"
    fi
  done
  ```
  Note: plain `yamllint -c .yamllint <file>` alone is **not** sufficient for the audit trio, since they contain top-level Jinja macros that plain YAML misparses as a syntax error. The command above replicates the project's actual `ci_lint.yml` logic, which strips Jinja constructs before linting.
  ![yamllint output](dod_5.JPG)

- [x] `./build_product rhel9 --datastream` completes without errors
  ```bash
  ./build_product rhel9 --datastream
  ```
  ![build output](dod_2.JPG)

**Run order:** the grep check (DoD removed) must pass before the lint/build checks are meaningful, since lint and build do not validate wording content.

### Manual Testing

Manually re-read each of the five modified files after editing to confirm the YAML structure is valid, the field order is unchanged, and the rewritten prose preserves the original security rationale.

---

## Implementation Notes

### Week 1 Progress

Cloned the repository and located all six candidate files referenced in the maintainer's grep output. Read full file contents (not just the grep snippets) to understand the complete context around each DoD mention. Drafted before/after wording for five files. Identified that `mcafee_security_software/group.yml` is fundamentally different from the other five, since its entire content is a DoD-specific mandate with no generic equivalent, and set it aside to avoid misrepresenting the rule's purpose. Drafted a scoping comment for the issue, naming exactly which files this PR would cover, to avoid the ambiguity that caused contribution #1's PR to be closed; posting it to the live issue is a remaining step before opening the PR.

### Code Changes

- **Files modified:** `audit_rules_execution_semanage/rule.yml` (DoD wording + one-line trailing-whitespace cleanup), `audit_rules_execution_setfiles/rule.yml`, `audit_rules_execution_setsebool/rule.yml`, `httpd_antivirus_scan_uploads/rule.yml`, `install_hids/rule.yml`
- **Key commits:** [To be added]
- **Approach decisions:** Kept edits to a single sentence or paragraph per file rather than rewording surrounding context, to keep the diff minimal and reviewable. Left `mcafee_security_software/group.yml` out of this PR rather than guessing at a generalized version of a DoD-mandated product requirement. Included the trailing-whitespace fix on `audit_rules_execution_semanage/rule.yml` in this PR rather than opening a separate one, since it's a one-character fix on a file already being touched and falls under CONTRIBUTING.md's exception for small textual changes that don't shift meaning.

---

## Pull Request

**PR Link:**  https://github.com/ComplianceAsCode/content/pull/14834

**PR Description:**  Remove DoD-specific verbiage from rule.yml files#14834

**Maintainer Feedback:**
- [Pending]

**Status:** Not yet opened. Scoping comment drafted (not yet confirmed and posted to the live issue); edits drafted; local lint/build verification pending.

---

## Learnings & Reflections

### Technical Skills Gained

Learned to distinguish between DoD wording that's incidental to a generic security requirement and DoD wording that constitutes the entire substance of a rule, as with `mcafee_security_software`. This distinction, not just whether the text mentions DoD, is what determines whether a file belongs in this kind of PR.

### Challenges Overcome

The original issue's grep output only showed a couple of lines of context per match, which wasn't enough to safely edit any of the files without first reading them in full. Some DoD mentions turned out to be load-bearing (banner text matched by a check, a McAfee-mandate rule) rather than incidental, and that distinction wasn't visible from the grep snippet alone.

Running plain `yamllint` on the three audit rule files produced a syntax error that initially appeared to be caused by the edit. Tracing it back to the original, unedited file confirmed it was pre-existing, caused by a top-level Jinja macro that plain YAML cannot parse on its own. Checking the project's own `ci_lint.yml` workflow confirmed this is expected and already handled in CI via `utils/strip_jinja_for_yamllint.py`, which strips Jinja constructs before linting. This was a useful reminder to verify against the project's actual CI logic rather than assuming a generic tool's output applies directly.

### What I'd Do Differently Next Time

Pull the full file content before triaging matches into "easy/medium/hard" buckets, rather than triaging from grep snippets first. A couple of files initially looked like easy wins from the snippet alone, but turned out to require more judgment once the full file was visible.

---

## Resources Used

- [ComplianceAsCode/content issue #8709](https://github.com/ComplianceAsCode/content/issues/8709)
- [ComplianceAsCode/content PR #13622 (prior PR covering part of this issue; maintainer confirmed direction, merge status not independently verified)](https://github.com/ComplianceAsCode/content/pull/13622)
- [ComplianceAsCode Style Guide](https://complianceascode.readthedocs.io/en/latest/manual/developer/04_style_guide.html)
- [ComplianceAsCode Contributing Guidelines (CONTRIBUTING.md)](https://github.com/ComplianceAsCode/content/blob/master/CONTRIBUTING.md)
- [ComplianceAsCode Developer Guide](https://complianceascode.readthedocs.io/)
