# Contribution #2: Remove DoD Specific Verbiage from rule.yml Files

**Contribution Number:** 2

**Student:** Jonathan Morales

**Issue:** [https://github.com/ComplianceAsCode/content/issues/8709](https://github.com/ComplianceAsCode/content/issues/8709)

**Status:** Phase I, In Progress

---

## Why I Chose This Issue

After contribution #1 closed, I wanted a second issue in which the design direction had already been validated by a maintainer, rather than one I'd have to resolve myself mid-PR. Issue #8709 fit that: it carries the "good first issue" label, a prior contributor's partial fix was already merged and confirmed as correct by the maintainer, and on October 9, 2025, the maintainer posted a fresh grep of all remaining DoD-specific text across the codebase, showing 119 occurrences across 37 files and explicitly inviting further triage. That gave me a concrete, maintainer-approved worklist to draw from instead of an open question, which was the gap that caused #1 to stall.

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

The DoD-specific wording in the five in-scope files falls into two patterns: boilerplate narrative text copied across multiple audit rules that frames a generic audit capability as a DoD requirement, and named DoD-specific entities (a form number, a product name, a government contact URL) standing in for a generic process or capability that any organization could substitute its own equivalent for.

### Proposed Solution

Reword each occurrence to describe the underlying security requirement without DoD-specific framing, while preserving the technical content and intent of the original text. No XCCDF variable changes are needed for this batch, since none of the affected text resolves to a profile-specific value; it's prose describing rationale, not a configurable setting.

### Fix Approach (How It Was Solved)

Edits were applied in three steps, in this order:

**Step 1: the three audit rule files (identical single-line change in each), via `sed`:**

```bash
sed -i 's/DoD has defined the list of events for which the operating system will provide an audit record generation capability as the following:/The list of events for which the operating system must provide an audit record generation capability includes the following:/' \
  ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml \
  ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setfiles/rule.yml \
  ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setsebool/rule.yml
```

**Step 2: the two multi-line/multi-sentence files, via a Python exact-match replace** (chosen over `sed` because the original text spans multiple lines with parentheses, which `sed` cannot match cleanly):

```bash
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
        print(f"NOT FOUND in {edit['path']} -- no changes made")
        continue
    p.write_text(text.replace(edit["old"], edit["new"]))
    print(f"Updated {edit['path']}")
PYEOF
```

**Step 3: an incidental trailing-whitespace cleanup**, found while lint-checking `audit_rules_execution_semanage/rule.yml` (see "Pre-Existing Lint Issue" below). Since this PR already modifies that file, the one-character fix was made in the same PR rather than left for a separate one:

```bash
sed -i '30s/[ \t]*$//' linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml
```

After each step, the change was verified with:

```bash
git diff
```

to confirm only the intended lines changed and no whitespace or indentation was disturbed elsewhere.

### Pre-Existing Lint Issue Discovered (Not Caused by This PR)

Running plain `yamllint` against the three audit rule files initially produced a `syntax error: could not find expected ':'`. Investigation traced this to a top-level Jinja macro call (`{{{ ocil_fix_srg_privileged_command(...) }}}`) that plain YAML misreads as the start of a flow-mapping. This was confirmed to be **pre-existing and unrelated to this PR** by parsing the original, unedited file directly: the same error occurs at the same line before any DoD-wording edit is applied.

The project's own CI (`.github/workflows/ci_lint.yml`) already accounts for this: any file containing Jinja constructs is piped through `utils/strip_jinja_for_yamllint.py` before linting, rather than linted directly. Reproducing that exact CI logic locally:

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

- [ ] Project's CI-equivalent lint check passes on all five modified files
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
  ![yamllint output](yamllint_dod.JPG)

- [ ] `./build_product rhel9 --datastream` completes without errors
  ```bash
  ./build_product rhel9 --datastream
  ```
  ![build output](build_dod.JPG)

- [ ] No DoD references remain in the five modified files
  ```bash
  grep -n "DoD" \
    ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_semanage/rule.yml \
    ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setfiles/rule.yml \
    ./linux_os/guide/auditing/auditd_configure_rules/audit_execution_selinux_commands/audit_rules_execution_setsebool/rule.yml \
    ./linux_os/guide/services/http/securing_httpd/httpd_configure_os_protect_web_server/httpd_antivirus_scan_uploads/rule.yml \
    ./linux_os/guide/system/software/integrity/endpoint_security_software/install_hids/rule.yml
  ```
  (expected: no output, confirming all DoD references were removed)
  ![after fix grep confirmation](fix_dod.JPG)

### Manual Testing

Manually re-read each of the five modified files after editing to confirm the YAML structure is valid, the field order is unchanged, and the rewritten prose preserves the original security rationale.

---

## Implementation Notes

### Week 1 Progress

Cloned the repository and located all six candidate files referenced in the maintainer's grep output. Read full file contents (not just the grep snippets) to understand the complete context around each DoD mention. Drafted before/after wording for five files. Identified that `mcafee_security_software/group.yml` is fundamentally different from the other five, since its entire content is a DoD-specific mandate with no generic equivalent, and set it aside to avoid misrepresenting the rule's purpose. Posted a scoping comment on the issue before starting edits, naming exactly which files this PR would cover, to avoid the ambiguity that caused contribution #1's PR to be closed.

### Code Changes

- **Files modified:** `audit_rules_execution_semanage/rule.yml` (DoD wording + one-line trailing-whitespace cleanup), `audit_rules_execution_setfiles/rule.yml`, `audit_rules_execution_setsebool/rule.yml`, `httpd_antivirus_scan_uploads/rule.yml`, `install_hids/rule.yml`
- **Key commits:** [To be added]
- **Approach decisions:** Kept edits to a single sentence or paragraph per file rather than rewording surrounding context, to keep the diff minimal and reviewable. Left `mcafee_security_software/group.yml` out of this PR rather than guessing at a generalized version of a DoD-mandated product requirement. Included the trailing-whitespace fix on `audit_rules_execution_semanage/rule.yml` in this PR rather than opening a separate one, since it's a one-character fix on a file already being touched and falls under CONTRIBUTING.md's exception for small textual changes that don't shift meaning.

---

## Pull Request

**PR Link:** [To be submitted]

**PR Description:** [To be drafted]

**Maintainer Feedback:**
- [Pending]

**Status:** Not yet opened. Comment posted on issue scoping this subset, edits drafted, local lint/build verification pending.

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
- [ComplianceAsCode/content PR #13622 (merged precedent)](https://github.com/ComplianceAsCode/content/pull/13622)
- [ComplianceAsCode Style Guide](https://complianceascode.readthedocs.io/en/latest/manual/developer/04_style_guide.html)
- [ComplianceAsCode Contributing Guidelines (CONTRIBUTING.md)](https://github.com/ComplianceAsCode/content/blob/master/CONTRIBUTING.md)
- [ComplianceAsCode Developer Guide](https://complianceascode.readthedocs.io/)
