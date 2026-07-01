---
name: pvm-forward-port
description: This skill should be used when the user asks to "forward-port PVM", "rebase PVM onto a newer kernel", "port the Pagetable-based Virtual Machine to <version>", "continue the PVM mainline port", "update kvm-pvm for Linux 7.x", or otherwise move the out-of-tree PVM hypervisor (virt-pvm/linux) from an older base onto a newer mainline Linux release. It encodes the incremental-hop method, the 6.12→7.1 API-drift fix catalog, the required config/build steps, the two runtime bugs, the neo-FSGSBASE/RDTSCP support, the spectre_v2 int3 fix, and the crash-debugging techniques learned porting PVM from 6.12.33 to 7.1.
version: 1.1.0
---

# Forward-porting PVM onto mainline Linux

PVM (Pagetable-based Virtual Machine, `github.com/virt-pvm/linux`) is an out-of-tree KVM-style
hypervisor that runs guests **without hardware virtualization** via a software "switcher" + shadow
MMU, exposing `/dev/kvm`. It is **131 commits on Linux 6.12.33**, living in the two churniest parts
of the kernel — the x86 **entry** asm and the **KVM x86** core. A forward-port is therefore *not*
"apply a patch"; it is **textual conflict resolution + compile-driven API-drift repair + runtime
debugging**. This skill captures the method and the concrete fixes proven from the 6.12→7.1 port.

Reference material (load on demand — do not read all up front):
- `references/method.md` — the incremental-hop procedure + conflict taxonomy (read first for any port)
- `references/api-drift-catalog.md` — concrete 6.12→7.1 symbol/signature changes and their fixes
- `references/runtime-fixes.md` — the two runtime bugs + no-FSGSBASE/RDTSCP + spectre_v2 int3
- `references/debugging.md` — crash-survivable debugging when printk is dead
- `references/build-validate-operate.md` — config, build, microVM test, and host-safety discipline
- `scripts/hop.sh` — a per-hop rebase driver you can adapt

## CPU / architecture context (get this right before touching FSGSBASE or mitigations)

The FSGSBASE/RDTSCP and Spectre work is **conditional on the host CPU**, not universal. Do not
hard-code these fixes; gate them and re-derive per host.

- **FSGSBASE and RDTSCP are x86-64 ISA extensions present on modern Intel *and* AMD parts** — PVM's
  dependency on them is **not** vendor-specific. The problem is **feature masking on specific
  (usually nested/emulated) instances**, not the vendor.
- Hosts this port actually ran on: **AMD EPYC Bergamo (4th gen), AMD EPYC Milan (3rd gen), Intel Xeon
  Platinum 8255C, Intel Xeon E5-2667 v4.** The EPYC and Xeon Platinum hosts **exposed** FSGSBASE, so
  PVM's fast `RD/WRGSBASE` switcher path runs unchanged and the no-FSGSBASE/RDTSCP fallback series is
  **inert** there (it is `ALTERNATIVE`-gated on `X86_FEATURE_FSGSBASE`). Only the **nested Intel Xeon
  E5-2667 v4 under Bochs BIOS** had FSGSBASE **and** RDTSCP masked → that host *needs* the fallback
  series (`references/runtime-fixes.md`). Bottom line: on a feature-complete host (any recent EPYC or
  Xeon), the FSGSBASE/RDTSCP patches cost nothing and change no behavior.
- The **`int3` / RSB finding is micro-architecture-dependent.** It is the RSB-VMEXIT
  (`FILL_RETURN_BUFFER`) mitigation colliding with a **nested** CPU's *non-architectural* RSB-underflow
  behavior — which is hypervisor-defined, and Intel vs AMD differ in RSB / Spectre-v2 / retbleed
  behavior. `spectre_v2=off` was the correct knob **on the nested Bochs target**; on a different host
  CPU the int3 may not reproduce at all, or a different knob may apply. Diagnose it per host, don't
  assume (`references/debugging.md` has the cross-check method).
- **Build always disables both vendor modules** (`-d KVM_INTEL -d KVM_AMD`) — PVM replaces them — but
  several kernel paths are gated on `CONFIG_KVM_INTEL`; extend those to `|| CONFIG_KVM_PVM` (the FRED
  sites and `x86_entry_from_kvm`; see the catalog).

## The one rule that matters most

**Incremental, one stable release at a time, and BUILD at every hop.** A direct 6.14→7.1 heuristic
rebase is lossy: take-ours/union auto-resolvers silently *drop* PVM additions on shared files and
*duplicate* others — it applies clean, won't compile, and worst of all *compiles but misbehaves*.
Do `6.12 → 6.13 → 6.14 → … → 7.1`, resolving + compiling each hop. The compile step is the whole
game: most breakage is **silent API drift** (the patch applies but a signature moved underneath it).

## Procedure (summary — full detail in `references/method.md`)

1. **Set up the tree.** Blobless clone of `virt-pvm/linux` + a `torvalds` remote; fetch the target
   tags. Work in `~/pvm-port`. Enable `git rerere`.
2. **Rebase one hop:** `git rebase --onto vN <pvm-base-sha> <prev-branch>` into `port-vN`. Drop
   commits already upstream.
3. **Resolve by conflict class** (taxonomy in `references/method.md`): scaffolding
   (PIE/objtool/percpu/relocation/stackprotector) → take upstream; KVM API headers → union/both;
   PVM-core files → hand-merge; boot/PIE → keep PVM. Never run two git drivers at once — it corrupts
   `.git/index` ("index file smaller than expected"); recover with `rm .git/index && git read-tree HEAD`.
4. **Build and fix drift.** `make -j bzImage modules` with `x86_64_defconfig + CONFIG_KVM_PVM=m +
   CONFIG_X86_FRED=y` (vendor KVM off). Fix each compile/link error using `references/api-drift-catalog.md`.
5. **Restore dropped PVM pieces.** Take-ours resolution silently drops exports, Makefile lines, and
   asm-offsets. The catalog lists the ones that bit us (switcher objects, `entry_64.S` hooks,
   asm-offsets, EXPORT lines).
6. **Runtime-validate** (see `references/build-validate-operate.md`): QEMU-TCG boot for a
   no-FSGSBASE host, then the Firecracker microVM test → expect `RESULT_GUEST_UP=1`.
7. **Apply the runtime fixes** from `references/runtime-fixes.md` (MSR-hypercall routing + host
   `task_pt_regs` restore) — these are NOT conflicts; they only surface when a guest actually runs.

## Hard-won landmines (each cost many reboots)

- **7.1 needs `CONFIG_X86_FRED=y`** even without FRED hardware — the KVM x86-64 external-interrupt
  path routes through a *software* FRED dispatcher. Gate its 6 `#if IS_ENABLED(CONFIG_KVM_INTEL)`
  sites with `|| IS_ENABLED(CONFIG_KVM_PVM)`.
- **7.1 namespaced KVM exports** — with only `KVM_PVM=m`, vmlinux emits *zero* KVM exports. Register
  `kvm-pvm` as a 3rd vendor sub-module in `arch/x86/include/asm/kvm_types.h`. (catalog §export)
- **The `int3` host crash is a mitigation, not a bug** — boot the host with **`spectre_v2=off`**
  (keeps MDS/TAA/MMIO/L1TF; strictly better than `mitigations=off`). See `references/runtime-fixes.md`.
- **7.1's `load_seg_legacy()` FS-base caching silently corrupts guest TLS** — 7.1 skips writing
  `MSR_FS_BASE` when `next_base == 0`, but PVM sets the base via hypercall (no hardware awareness), so
  the host keeps a stale FS base → thread-local storage corruption. Force the write for PVM guests in
  `arch/x86/kernel/process_64.c` (catalog §process_64). This is a *runtime* corruption, not a compile
  error — easy to miss.
- **`dracut --force` after every module change** — the initramfs bundles `kvm-pvm.ko`; otherwise the
  boot loads the STALE module. Verify with `modinfo -F srcversion` of loaded vs installed.
- **Keep the working old kernel as the GRUB `saved_entry`** with `panic=5` so a crash auto-reverts;
  test the new kernel with a one-time `grub2-reboot`. Restore any production services (e.g.
  `hive-node`) + reload `kvm-pvm` before leaving the host on the old kernel.

Full operational discipline (host-safety, kdump, module refcount, reboot etiquette) is in
`references/build-validate-operate.md`. When continuing a specific in-flight port, also check the
project's memory notes (e.g. `pvm-mainline-forward-port.md`) for the current branch and last state.

---
*Provenance: every fix, symbol, and CPU detail here is drawn from the actual 6.12→7.1 port —
`pvm-6.12-to-7.1-complete.patch` (the 110-commit series) and the write-up "Anatomy of a 9-hop kernel
rebase: bringing PVM to Linux 7.1". Nothing here is inferred; if a future hop contradicts a detail,
trust the compiler/kdump over this doc and update it.*
