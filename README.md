# Contribution #1: Label the TLS in vmmap

**Contribution Number:** 1

**Student:** Jonathan Morales

**Issue:** [https://github.com/pwndbg/pwndbg/issues/1570](https://github.com/pwndbg/pwndbg/issues/1570)

**Status:** Phase I

---

## Why I Chose This Issue

This issue is a focused and well-scoped enhancement inside pwndbg's memory inspection tooling. The `vmmap` command already labels regions like `[heap]` and `[stack]`, but the TLS region is left anonymous even though its address is already known. That gap makes TLS harder to spot during a debugging session, which matters in exploit development and reverse engineering workflows.

The fix touches a small, well-defined part of the codebase and requires understanding how pwndbg builds its memory map, how the `tls` command resolves addresses, and how the `Page` object controls what labels are displayed.

---

## Understanding the Issue

### Problem Description

The `tls` command resolves the Thread Local Storage base address using architecture-specific registers (`fsbase` on x86-64). The `vmmap` command has no awareness of this and displays the same memory region with a generic anonymous label instead of `[tls]`.

### Expected Behavior

The TLS memory region should be labeled `[tls]` in `vmmap` output, consistent with how `[heap]` and `[stack]` are already annotated.

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

- `pwndbg/commands/tls.py` - resolves TLS base address
- `pwndbg/aglib/vmmap.py` - fetches and returns memory map pages
- `pwndbg/lib/memory.py` - defines the `Page` object and its `objfile` field

---

## Reproduction Process

### Environment Setup

Reproduced on Ubuntu 24.04 running inside a VMware virtual machine. pwndbg was cloned from the official repository and installed via `./setup.sh`. A missing C++ compiler caused the setup to fail initially, resolved by running `apt install -y g++ build-essential` before retrying.

### Steps to Reproduce

1. Launch GDB targeting `/usr/bin/sleep`:
   ```
   gdb /usr/bin/sleep
   ```
2. Stop at the very first instruction before anything initializes:
   ```
   starti 30
   ```
3. Set a catchpoint for libc to wait until the dynamic linker loads it:
   ```
   catch load libc
   continue
   ```
4. Continue again and interrupt the process once it is sleeping:
   ```
   continue
   ^C
   ```
5. Run `tls` and note the resolved base address:
   ```
   tls
   ```
6. Run `vmmap` and find the same address range labeled as anonymous:
   ```
   vmmap
   ```

### Reproduction Evidence

**Step 2 - Stopped at `_start` before TLS is initialized:**

![starti output](starti_1.JPG)

**Step 3 - Caught libc loading via the dynamic linker:**

![catch load libc output](catch_load_libc_2.JPG)

**Step 4 - Process interrupted inside `clock_nanosleep` while sleeping:**

![continue and Ctrl+C output](continue_cntr.JPG)

**Steps 5 and 6 - `tls` resolves the base address, `vmmap` shows the same region as anonymous:**

![tls output showing anon label](tls.JPG)

**Findings:** The `tls` command correctly identifies `0x7ffff7fa7740` as the TLS base and internally calls `pwndbg.aglib.vmmap.find(tls_base)` to locate the containing page. However that page label is never updated. The two commands resolve the same address independently with no shared labeling step between them.

---

## Solution Approach

### Analysis

The root cause is that `vmmap` builds its page list from `/proc/PID/maps`, which labels anonymous mappings generically. The `tls` command resolves the TLS address separately but does not write that information back into the memory map. The `Page` object has an `objfile` field that controls the label displayed in `vmmap` output. Setting that field to `[tls]` on the matching page is the correct fix.

### Proposed Solution

After the memory map pages are fetched, call `pwndbg.aglib.tls.find_address_with_register()` to get the TLS base address. Find the page that contains that address and set its `objfile` to `[tls]`. If TLS is not initialized or the address cannot be resolved, skip the labeling silently.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `vmmap` shows the TLS region as anonymous because it reads labels directly from the OS with no TLS awareness. The address is already resolvable through the existing `tls` command infrastructure.

**Match:** The `Page` object's `objfile` field is used throughout the codebase to control region labels. The fix follows the same pattern already used for other named regions.

**Plan:**
1. In `pwndbg/aglib/vmmap.py`, after pages are fetched, call `pwndbg.aglib.tls.find_address_with_register()`.
2. If the returned address is valid (non-zero), call `get_memory_map().lookup_page(tls_address)` to find the matching page.
3. If a page is found, set `page.objfile = "[tls]"`.
4. If TLS is not initialized or the address is 0, do nothing and return cleanly.
5. Add tests to verify the label appears correctly and existing labels are unaffected.

**Implement:** [Link to branch/commits as work progresses]

**Review:** Verify the change follows pwndbg contribution guidelines, does not break existing `vmmap` tests, and handles edge cases including uninitialized TLS, remote targets, and non-x86 architectures.

**Evaluate:** Run `vmmap` after interrupting a live process and confirm the TLS region displays as `[tls]`.

---

## Testing Strategy

### Unit Tests

- [ ] TLS region is labeled `[tls]` when the address is valid and resolved via register
- [ ] No label change occurs when TLS address cannot be resolved (returns 0 or None)
- [ ] Existing `[heap]` and `[stack]` labels are unaffected

### Integration Tests

- [ ] Full reproduction scenario: `starti` then `continue` then `Ctrl+C` then `vmmap` shows `[tls]`
- [ ] Behavior is correct when multiple threads are present

### Manual Testing

Reproduced the bug on Ubuntu 24.04 with `/usr/bin/sleep 30` as the target. After the process was fully initialized and interrupted, `tls` resolved the address and `vmmap` confirmed the same region is labeled `[anon_...]`. Fix not yet implemented.

---

## Implementation Notes

### Week 1 Progress

Reproduced the issue on a local Ubuntu VM. Read through `tls.py` and `vmmap.py` to understand how pages are fetched and how `objfile` controls labeling. Confirmed the `Page` object is the right place to apply the fix. Setup required installing a missing C++ compiler before `./setup.sh` could complete.

### Code Changes

- **Files to modify:** `pwndbg/aglib/vmmap.py`
- **Key commits:** [To be added]
- **Approach decisions:** Using register-based TLS resolution only via `find_address_with_register()` to avoid the implicit `pthread_self()` call concerns raised in the issue thread.

---

## Pull Request

**PR Link:** [To be submitted]

**PR Description:** [To be drafted]

**Maintainer Feedback:** [Pending]

**Status:** In progress

---

## Learnings and Reflections

### Technical Skills Gained

Gained familiarity with how pwndbg builds and labels memory map pages, and how the `tls` command resolves addresses through architecture-specific registers. Better understanding of TLS initialization order relative to the dynamic linker.

### Challenges Overcome

TLS is not available at `_start` so early breakpoints returned nothing. The correct approach was letting the program fully initialize via `continue` and then interrupting it with `Ctrl+C`. Initial attempts with `break main` failed because the binary had no debug symbols.

### What I Would Do Differently Next Time

Read the relevant source files before starting reproduction. Understanding `objfile` and how page labels work earlier would have made the fix direction clear from the start.

---

## Resources Used

- [pwndbg issue #1570](https://github.com/pwndbg/pwndbg/issues/1570)
- [pwndbg source - aglib/vmmap.py](https://github.com/pwndbg/pwndbg/blob/dev/pwndbg/aglib/vmmap.py)
- [pwndbg source - commands/tls.py](https://github.com/pwndbg/pwndbg/blob/dev/pwndbg/commands/tls.py)
- [pwndbg official documentation](https://pwndbg.re)
