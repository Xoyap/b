# 0xDEAD OS architecture and low-rewrite migration plan

## Chosen direction

Use a **modular monolithic kernel first**, while keeping service boundaries suitable for later user-space services and a possible microkernel migration. The short-term objective is a useful, visibly progressing OS without coupling every subsystem to kernel internals.

## Non-negotiable engineering rules

1. Preserve a known-good source/build and test baseline before each behavioral change.
2. Give each subsystem one owner and a narrow documented interface.
3. Keep mechanism separate from policy (for example, framebuffer drawing primitives from desktop/window policy, and block I/O from FAT/VFS).
4. Do not expose kernel-private pointers as user-facing ABI. Use validated fixed-width requests and opaque IDs/handles across future process/service boundaries.
5. Keep kernel panic/serial diagnostics independent from the graphical UI.
6. Introduce abstractions when they create an actual boundary; avoid a broad speculative rewrite.
7. Move services one at a time only after isolation, IPC, lifetime handling, error recovery and tests are in place.

## Current observations from `os.asm`

- UEFI boot code transfers control to `kernel_entry`; the current shell and desktop renderer are still kernel-side.
- `kernel_entry` calls `desktop_draw`. The first migration step introduces `ui_backend_draw` as a behavior-preserving internal seam; it is still Ring 0 and is not a userspace implementation.
- `process_create` allocates a private PML4 root, then copies the current kernel PML4 entries into it and marks the address space ready. A private root alone does **not** prove Ring 3 isolation. User-mode execution must wait until effective page permissions, user mappings, kernel supervisor-only mappings, transition stacks, exception handling and process cleanup are validated.
- No syscall or IPC implementation was identified in this source during this review. Do not treat the existing process descriptors as proof that user-mode execution already works.

These observations are source review only; they are not a claim that this revision builds or passes QEMU tests.

## Target subsystem boundaries

| Subsystem | Stable contract to establish | Future direction |
|---|---|---|
| Boot / UEFI | Explicit boot information and one handoff contract | Keep boot stage small; kernel must not depend on Boot Services after exit |
| Physical / virtual memory | Allocation, mapping, unmapping, translation and permission APIs | Core-kernel mechanism |
| CPU / interrupts | Architecture entry points, interrupt-safe contracts, panic path | Core-kernel mechanism |
| Threads / scheduler | Create, run, block, wake, yield and exit contracts | Core-kernel mechanism |
| Process / address spaces | Process lifecycle and controlled address-space operations | Core mechanism; policy and permissions need careful verification |
| UI / graphics | Renderer primitives separated from layout/window policy | Keep a recovery console; move the compositor/UI to user space later |
| Storage / filesystem | Block requests separated from VFS/filesystem semantics | Filesystem can become a user-space service |
| Syscalls / IPC | Versioned fixed-width ABI, validated pointers, opaque handles and explicit errors | Controlled boundary for user processes and services |
| Diagnostics | Stable logs and structured error/panic reports | Must work without the desktop |

## Ordered roadmap

1. **Freeze baseline:** record the exact source revision, build command, QEMU configuration, full logs and current test counts. Keep the known-good 27-pass revision recoverable.
2. **Add safe seams:** make one small interface change at a time. This PR adds only the UI backend seam and this document.
3. **Prove Ring 3 isolation:** audit GDT/TSS/IDT transitions, build user code/data/stack mappings, ensure kernel pages are supervisor-only at every paging level, and add a minimal user-mode fault-containment test.
4. **Define syscall ABI:** establish syscall numbers, argument/return/error conventions and pointer validation before adding multiple services.
5. **Add IPC carefully:** define endpoint/handle lifetime, message sizes, validation, blocking/wakeup, dead-peer and allocation-failure behavior.
6. **Move one component at a time:** shell/diagnostics, GUI/compositor, filesystem, then suitable drivers/network services. Keep panic and serial output kernel-side.
7. **Re-evaluate architecture:** migrate only components that benefit; do not promise a full microkernel without measured reasons.

## Acceptance gate for each code change

- Assemble with the existing local NASM/link command.
- Boot in the same QEMU setup.
- Existing tests retain their prior pass count; no regression is silently accepted.
- New behavior has a focused test and understandable failure reporting.
- Preserve a known-good source and EFI image for rollback.
