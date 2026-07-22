# Contribution #3: Remove DoD Specific Verbiage from rule.yml Files (Part 2)

**Contribution Number:** 3
**Student:** Jonathan Morales
**Issue:** [https://github.com/ComplianceAsCode/content/issues/8709](https://github.com/ComplianceAsCode/content/issues/8709)
**Status:** Phase IV Complete — PR Merged

---

## Pull Request

**PR Link:** [https://github.com/ComplianceAsCode/content/pull/14841](https://github.com/ComplianceAsCode/content/pull/14841)

**PR Description:**

*What does this PR do?* Continues addressing issue #8709 by removing or generalizing DoD-specific phrasing across 15 additional `rule.yml`/`.var` files, following the pattern established by the prior merged PR (#14834). Keeps DoD wording where it mirrors upstream STIG content or documents a requirement that is intentionally DoD-specific, and generalizes it only where it was incidental prose.

*Why was this PR needed?* Issue #8709 asks that security content describe requirements in policy-agnostic language where the DoD reference is incidental, so the same content applies to any organization using the benchmark, not only DoD. The maintainer's October 2025 grep of the codebase surfaced 119 remaining occurrences across 37 files; this PR covers 15 of the remaining files after the first 5 were merged in #14834.

*What are the relevant issue numbers?* Updates #8709 (does not close it; the issue is intentionally left open for further contributions across the remaining files, per maintainer confirmation on the original PR).

*Does this PR meet the acceptance criteria?*
- [x] Follows the project's style guide for prose edits (no field reordering, no logic changes)
- [x] No breaking changes to OVAL checks, remediations, or rule structure
- [x] Existing tests unaffected
- [ ] New automated test added — not applicable to most of this PR (prose-only edits); one test scenario was attempted for the `generic_default` variable option but removed after confirming a genuine test-framework limitation, documented below
- [x] All maintainer feedback addressed and merged

---

## Maintainer Feedback

- **Round 1:** Maintainer (Mab879) requested reverting 8 STIG-specific files back to original DoD wording, and removing the required banner text entirely (not just rewording the lead-in sentence) from 4 banner files.
  **Response (commit `5f56e413f1`):** Reverted the 8 files to original upstream text, verified with direct diffs against `upstream/master`. Removed the required-text block from 3 of 4 banner files. Left the 4th, `banner_etc_profiled_ssh_confirm/rule.yml`, unchanged and posted a follow-up question, since its DoD text is embedded in a shell script rather than a standalone block.

- **Round 2:** Maintainer suggested using the existing `var_ssh_confirm_text` XCCDF variable instead of hardcoded text in `banner_etc_profiled_ssh_confirm/rule.yml`.
  **Response (commit `5e7e1a4c7f`):** Investigated the variable directly and found it already existed with one option (`dod_default`); added a second option, `generic_default`, generated with the project's own regex tool, and updated the rule's description to reference the variable.

- **Round 3:** Attempted to add an automated test for the new option across several commits (`6c3831823b`, `4498786d45`, `299096d454`, `0d48f7f3fc`), each addressing a CI failure in turn (metadata syntax, wording mismatch, Jinja parsing error from an unescaped apostrophe). After confirming via the actual test framework source that the underlying `variables` metadata mechanism cannot resolve a named option to a regex value, reported this finding with the specific code reference.
  **Maintainer response:** Confirmed the limitation and offered to test manually instead.
  **Response (commit `1568131f4f`):** Removed the test scenario per that direction.

- **Round 4:** Maintainer did manual testing and found two further issues in the `generic_default` regex: an unescaped apostrophe causing remediation errors, and a core bug in the shared `escape_regex` utility where `?` was not being escaped and was interpreted as a regex quantifier.
  **Response:** Reworded the banner text to remove the apostrophe entirely (commit `c03651a812`). Applied the maintainer's own tested patch to `ssg/utils.py` exactly as supplied, and regenerated the `generic_default` regex with the corrected tool (commit `ede308b220`).

- **Round 5:** Maintainer noted one last nit: the `srg_requirement` field in `configure_gnutls_tls_crypto_policy/rule.yml` should have stayed verbatim, since STIG SRG fields mirror the official SRG text exactly, unlike the other fields in that file which were correctly generalized.
  **Response:** Reverted just that field back to the original DoD wording, verified against upstream with a direct diff. Commit hash pending confirmation from final push.

- **CI infrastructure noise:** Over the course of this PR, several CI checks failed for reasons unrelated to the content changes: an AWS load balancer quota limit on the CI provider's account (`TooManyLoadBalancers`) affecting the OpenShift e2e compliance jobs on multiple separate runs, a stuck `MachineConfigPools` node rollout timeout on another run, and a `stable-products` test failure on the unrelated `sle16` product caused by drift on `master` itself (independently confirmed unrelated, since this PR touches no SLE16 files, and later fixed by a separate contributor's PR #14875). These were flagged to the maintainer as unrelated rather than worked around, consistent with the project's own contribution guidelines on waivable CI failures.

**PR Status:** Merged.

---

## Learnings & Reflections

### Technical Skills Gained

Learned to trace a CI failure back to the actual framework source code that produces it, rather than stopping at a plausible-sounding explanation. The `variables` test-metadata investigation in particular required reading `tests/ssg_test_suite/rule.py` directly to find the exact function (`generate_xslt_change_value_template`) responsible, which turned a repeating cycle of guesses into a definitive, reportable finding. Also learned the practical difference between a regex escaping bug that breaks a *pattern match* (tolerable, since OVAL's check is a search, not exact equality) versus one that breaks *remediation* (not tolerable, since the deregexify step and shell interpolation are much stricter), which is why the same `?` character issue didn't surface until the maintainer's manual testing even though it had been present from an earlier point in the work.

### Challenges Overcome

The biggest challenge was resisting the pull to keep patching symptoms after each new CI failure without stepping back to find the actual root cause. Three consecutive fixes to the same test scenario each resolved one visible error only to expose the next, until reading the actual test-runner source code settled the question definitively instead of continuing to guess. A second real challenge was verifying an AI coding assistant's suggestions against the real codebase before applying them; more than one suggested fix during this contribution was plausible-sounding but incorrect once checked directly against working code already in the project.

### What I'd Do Differently Next Time

Read the relevant test framework source code earlier, before attempting to write a new test scenario for a variable option, rather than after three rounds of CI failures. Knowing upfront that the `variables` metadata mechanism does a literal substitution (not an option lookup) would have surfaced the limitation immediately instead of over several iterations.
