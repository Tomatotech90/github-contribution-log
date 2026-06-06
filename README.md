# Contribution #1: Label the TLS in vmmap

**Contribution Number:** 1

**Student:** Jonathan Morales

**Issue:** [https://github.com/pwndbg/pwndbg/issues/1570](https://github.com/pwndbg/pwndbg/issues/1570)

**Status:** Phase I

---

## Why I Chose This Issue

This issue sits at the intersection of memory management and debugger tooling, two areas central to low-level security work. The problem is well-scoped: `vmmap` already surfaces `[heap]` and `[stack]` as labeled regions, yet the TLS region — equally relevant during exploit development and reverse engineering — remains anonymous. That inconsistency makes TLS harder to identify quickly during a debugging session.

The fix requires understanding how pwndbg resolves memory regions, how the `tls` command works internally, and how the `Page` object's `objfile` field controls what gets displayed. It is a focused contribution that builds real familiarity with the pwndbg codebase without requiring changes across many systems.

---

## Understanding the Issue

### Problem Description

The `tls` command in pwndbg successfully resolves the Thread Local Storage base address using architecture-specific registers (`fsbase` on x86-64). However, `vmmap` has no awareness of this — the corresponding memory region is displayed with an anonymous label (`[anon_7ffff7fa7]`) rather than a meaningful one.

### Expected Behavior

The TLS memory region should appear labeled as `[tls]` in `vmmap` output, consistent with how `[heap]` and `[stack]` are already annotated.

### Current Behavior

```
pwndbg> tls
Thread Local Storage (TLS) base: 0x7ffff7fa7740
TLS is located at:
    0x7ffff7fa7000     0x7ffff7faa000 rw-p     3000       0 [anon_7ffff7fa7]

pwndbg> vmmap
...
    0x7ffff7fa7000     0x7ffff7faa000 rw-p     3000       0 [anon_7ffff7fa7]
...
```

### Affected Components

- `pwndbg/commands/tls.py` — resolves TLS base address
- `pwndbg/aglib/vmmap.py` — fetches and returns memory map pages
- `pwndbg/lib/memory.py` — defines the `Page` object and its `objfile` field

---

## Reproduction Process

### Environment Setup

Reproduced on Ubuntu 24.04 (VMware virtual machine). pwndbg was cloned from the official repository and installed via `./setup.sh`. One issue encountered during setup was a missing C++ compiler; resolved by running `apt install -y g++ build-essential` before retrying the install script.

### Steps to Reproduce

1. Launch GDB with pwndbg loaded targeting `/usr/bin/sleep`:
   ```
   gdb /usr/bin/sleep
   ```
2. Stop at the first instruction:
   ```
   starti 30
   ```
3. Set a catchpoint for libc and continue:
   ```
   catch load libc
   continue
   ```
4. Allow the program to fully initialize, then interrupt it:
   ```
   continue
   ^C
   ```
5. Run `tls` — note the resolved base address:
   ```
   tls
   ```
6. Run `vmmap` — locate the same address range and observe the anonymous label:
   ```
   vmmap
   ```

### Reproduction Evidence

**Step 2 — Stopped at `_start` before TLS is initialized:**

![starti output](starti_1.JPG)

**Step 3 — Caught libc loading via dynamic linker:**

![catch load libc output](catch_load_libc_2.JPG)

**Step 4 — Program interrupted mid-execution inside `clock_nanosleep`:**

![continue and Ctrl+C output](continue_cntr.JPG)

**Steps 5 & 6 — `tls` resolves the base, `vmmap` shows the region as anonymous:**

![tls output showing anon label](tls.JPG)

**My findings:** The `tls` command correctly identifies `0x7ffff7fa7740` as the TLS base and even calls `pwndbg.aglib.vmmap.find(tls_base)` internally to locate the containing page — yet that page's label is never updated. The two commands operate independently with no shared labeling step.

---

## Solution Approach

### Analysis

The root cause is that `vmmap` builds its page list from `/proc/PID/maps` (via the kernel), which labels anonymous mappings generically. The `tls` command resolves the TLS address separately but does not feed that information back into the memory map. The `Page` object exposes an `objfile` field that is already used to label regions like `[heap]`, `[stack]`, and `[vsyscall]` — the same mechanism can be used here.

### Proposed Solution

After the memory map pages are fetched, resolve the TLS base address using the existing `pwndbg.aglib.tls.find_address_with_register()` function. Find the page that contains that address and set its `objfile` to `[tls]`. This is consistent with how `[vsyscall]` labeling is already handled in the codebase.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `vmmap` displays the TLS region as anonymous because it reads labels directly from the OS without any TLS-awareness. The `tls` command already knows the address but does not share it with `vmmap`.

**Match:** In `pwndbg/aglib/vmmap.py`, the `[vsyscall]` region is labeled by finding the last page and setting `page.objfile = "[vsyscall]"`. The same pattern applies here.

**Plan:**
1. In `pwndbg/aglib/vmmap.py`, after pages are fetched, call `pwndbg.aglib.tls.find_address_with_register()` to get the TLS base.
2. Use `get_memory_map().lookup_page(tls_address)` to find the matching page.
3. If a page is found and the TLS address is valid, set `page.objfile = "[tls]"`.
4. Handle the case where TLS is not yet initialized (return value of 0 or `None`) gracefully without raising errors.
5. Update or add tests to verify the label appears correctly.

**Implement:** [Link to branch/commits as work progresses]

**Review:** Verify the change follows pwndbg's contribution guidelines, does not break existing `vmmap` tests, and handles edge cases (no TLS, remote targets, non-x86 architectures).

**Evaluate:** Run `vmmap` after interrupting a live process and confirm the TLS region displays as `[tls]`.

---

## Testing Strategy

### Unit Tests

- [ ] TLS region is labeled `[tls]` when TLS address is valid and resolved via register
- [ ] No label change occurs when TLS address cannot be resolved (returns 0 or None)
- [ ] Existing `[heap]` and `[stack]` labels are unaffected by the change

### Integration Tests

- [ ] Full reproduction scenario: `starti` → `continue` → `Ctrl+C` → `vmmap` shows `[tls]`
- [ ] Behavior is correct when multiple threads are present

### Manual Testing

Reproduced the bug manually on Ubuntu 24.04 with `/usr/bin/sleep 30` as the target. Confirmed that after the program is fully initialized, `tls` resolves the address and `vmmap` shows the same region as `[anon_...]`. Fix not yet implemented.

---

## Implementation Notes

### Week 1 Progress

Reproduced the issue on a local Ubuntu VM. Studied the relevant source files (`tls.py`, `vmmap.py`) to understand how pages are fetched and how `objfile` is used for labeling. Identified the `[vsyscall]` labeling pattern as the correct model for this fix. Setup challenges included a missing C++ toolchain during `./setup.sh`, resolved with `apt install -y g++ build-essential`.

### Code Changes

- **Files to modify:** `pwndbg/aglib/vmmap.py`
- **Key commits:** [To be added]
- **Approach decisions:** Using register-based TLS resolution only (`find_address_with_register`) to avoid the side-effect concerns raised in the issue thread around calling `pthread_self()` implicitly.

---

## Pull Request

**PR Link:** [To be submitted]

**PR Description:** [To be drafted]

**Maintainer Feedback:** [Pending]

**Status:** In progress

---

## Learnings & Reflections

### Technical Skills Gained

Gained hands-on familiarity with pwndbg's internal architecture, specifically how memory map pages are constructed, labeled, and consumed by commands. Deepened understanding of TLS initialization order relative to the dynamic linker and libc.

### Challenges Overcome

Reproducing the issue required understanding that TLS is not available at `_start` — the program must be running and fully initialized before `tls` can resolve the address. Initial attempts using `break main` failed because the binary had no debug symbols; `starti` followed by `continue` and `Ctrl+C` was the correct approach.

### What I'd Do Differently Next Time

Read the relevant source files before attempting reproduction, rather than in parallel. Understanding `objfile` and how `[vsyscall]` is labeled earlier would have clarified the fix direction sooner.

---

## Resources Used

- [pwndbg issue #1570](https://github.com/pwndbg/pwndbg/issues/1570)
- [pwndbg source — aglib/vmmap.py](https://github.com/pwndbg/pwndbg/blob/dev/pwndbg/aglib/vmmap.py)
- [pwndbg source — commands/tls.py](https://github.com/pwndbg/pwndbg/blob/dev/pwndbg/commands/tls.py)
- [pwndbg official documentation](https://pwndbg.re)
