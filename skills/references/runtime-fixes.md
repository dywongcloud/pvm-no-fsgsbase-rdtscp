# Runtime fixes and repo-integrity bugs found in the field

This file records concrete bugs found by actually building, diffing against
upstream, and researching this project's failure reports -- not inferred or
padded content. It exists because the `references/` directory promised by
`skills/README.md`'s "Layout" section was missing from this repository
entirely (only the top-level `SKILL.md`/`README.md` had been committed); this
is the first entry.

## 1. Case-insensitive `.gitignore` silently dropped ~1340 tracked files

**Symptom investigated:** hosts running "our upgrade PVM kernel" intermittently
freeze, reset, or become SSH-unreachable.

**Root cause:** `kernel/.gitignore` carries the standard upstream Linux rule
`*.s` (meant to ignore lowercase, compiler-generated preprocessed-assembly
output). Whoever populated this repository's `kernel/` tree did so from a
checkout with `core.ignorecase=true` (typical on macOS/Windows, or inherited
from a clone/template) -- under case-insensitive matching, `*.s` *also*
matches real, hand-written uppercase `.S` assembly source. This is a
well-documented, decades-old class of kernel/git gotcha (see LKML threads from
2004 and 2012, and Linus Torvalds' long-standing public objection to
case-insensitive/case-folding filesystems for exactly this reason).

The blast radius here was total: **1341 of 1344 `.S` files in the whole tree**
were absent from version control, including `arch/x86/entry/entry_64.S` (the
syscall/interrupt entry point) and, critically, **`arch/x86/entry/
entry_64_switcher.S` -- the PVM switcher itself**, the file this entire
project's patch series modifies. Only `tools/testing/selftests/kvm/` survived,
because its own local `.gitignore` happens to carry an explicit `!*.S`
override. (22 other files were separately missing for unrelated reasons --
upload-time case collisions on paths like `xt_DSCP.h`, unrelated to this
`.gitignore` rule.)

**Why this matters for host stability, not just repo hygiene:** a repository
that cannot even show you the file its own patch series is supposed to modify
cannot be the trustworthy source of record for what runs in production. Anyone
rebuilding from a fresh clone of this repo (rather than the original author's
local working tree, which *did* have the real files on disk, just invisible to
git) would either fail to build outright, or -- worse -- silently link against
a stale/mismatched object left over from a previous build. A GS-base/register
ABI mismatch between the switcher and the rest of PVM's C code is exactly the
kind of bug that produces intermittent, load-dependent host corruption rather
than a clean, always-reproducible failure.

**Fix:**
- Restored every missing file from `virt-pvm/linux@pvm-612` at the same base
  commit this tree derives from.
- Hardened `kernel/.gitignore` with an explicit `!*.S` (immediately
  re-ignoring the two genuinely-generated `*.dtb.S`/`*.dtbo.S` patterns that
  are supposed to match it), verified correct under both
  `core.ignorecase=true` and `=false`.
- Added `.github/workflows/gitignore-integrity.yml`, which forces
  `core.ignorecase=true` on the CI runner (so it reproduces the bug class even
  on a case-sensitive Linux runner) and runs `git ls-files -i -c
  --exclude-standard` -- the same technique the Linux kernel itself adopted in
  `scripts/misc-check` after hitting this same bug class in 2022.

## 2. The switcher's own FSGSBASE fallback (patch 1 of 4) was never actually applied

The README documents a 4-patch RFC series ("x86/PVM without FSGSBASE or
RDTSCP") as "verified end-to-end." Patches 2-4 (KVM emulator `em_rdtscp()`,
soft-gating `hardware_cap_check()`, and the guest-side `MSR_FS_BASE` hypercall
fallback in `arch/x86/kernel/pvm.c`) were present in the tracked tree. **Patch
1 -- the actual host switcher fallback in `entry_64_switcher.S` -- was not**,
because bug #1 above made that file invisible to git the entire time the
author was presumably validating and committing the others.

This is a materially dangerous half-applied state, not just an incomplete
one: patches 2-4 turn `hardware_cap_check()`'s FSGSBASE/RDTSCP checks from a
hard `-EOPNOTSUPP` (safe load-time refusal) into an informational `pr_info()`
(soft pass). Combined with a switcher that still executes `rdgsbase`/
`wrgsbase` unconditionally, this means `kvm-pvm` would **load successfully**
on exactly the target hardware class this whole project exists for (a host
where the outer hypervisor masks FSGSBASE), and then take a `#UD` on the very
first guest syscall or interrupt requiring a umod<->smod transition -- a
guaranteed-not-probabilistic crash storm on that class of host, which is a
strong, direct match for "sometimes boxes randomly freeze/reset/become
inaccessible via ssh" restricted to the subset of the fleet that actually
lacks FSGSBASE.

**Fix:** reconstructed patch 1 (ALTERNATIVE-gated MSR_KERNEL_GS_BASE fallback
for both switcher direct-switch paths) from the RFC's own published diff and
applied it to the now-restored `entry_64_switcher.S`. Verified by:
- Building the object under a real `x86_64_defconfig + CONFIG_KVM_PVM=m`
  config (vendor KVM disabled) -- compiles cleanly.
- Running `objtool --stackval --orc --retpoline --rethunk` against both the
  patched and unpatched object and diffing the warning sets -- **zero new
  warnings** (the one pre-existing `switcher_enter_guest+0xde: return with
  modified stack frame` warning is present in the unmodified upstream object
  too; not introduced by this change).
- Hand-tracing the GS-base/MSR_KERNEL_GS_BASE state machine for both the
  umod->smod and smod->umod directions to confirm the MSR-based fallback
  reaches the exact same architectural end state as the FSGSBASE fast path,
  and that the fallback introduces no new NMI-unsafe window (active GS.base
  stays host's for the *entire* MSR read/write sequence in the fallback path;
  only the trailing `swapgs` changes it, exactly mirroring the safety
  property of the fast path).

## 3. Unpatched, disclosed KVM shadow-MMU use-after-free (CVE-2026-53359, "Januscape")

Independent research turned up **CVE-2026-53359**, a CVSS 8.8 use-after-free
in KVM x86's shadow-paging code (a stale child shadow page can be reused
under a mismatched GFN/role), disclosed July 2026 and demonstrated as a
guest-triggered host crash / guest-to-host escape. It was fixed upstream on
2026-07-09 (`81ccda30b4e8` "KVM: x86: Fix shadow paging use-after-free due to
unexpected role", plus a companion "...unexpected GFN" fix) and backported
onto `virt-pvm/linux`'s `pvm-612` branch the same day.

**Directly confirmed against this repo:** diffing `kernel/arch/x86/kvm/mmu/
mmu.c` against the fixed upstream commit showed this repository's tree
predates the fix -- `kvm_mmu_get_child_sp()`'s existence check only tested
`is_shadow_present_pte() && !is_large_pte()` without verifying the *existing*
child's GFN/role actually matched what was being requested, and
`kvm_mmu_get_shadow_page()`'s "shadow page already present" path called the
old `drop_large_spte()` (a bare SPTE drop) instead of properly zapping the
mismatched child through `mmu_page_zap_pte()` + a remote TLB flush/zap. Since
PVM's shadow MMU shares this exact code path, this repo's tree carried a
**real, disclosed, guest-triggerable host memory-corruption bug** -- a very
strong candidate for "sometimes boxes randomly freeze/reset/become
inaccessible via SSH" under ordinary (non-malicious) guest workloads that
happen to exercise the same GFN/role-reuse pattern, since shadow-MMU UAF
corruption is exactly the class of bug that produces nondeterministic,
load-dependent host crashes rather than a clean, always-reproducible one.

**Fix:** took the fixed `mmu.c` from `virt-pvm/linux@pvm-612` wholesale (the
entire diff was this one coherent security fix, nothing else mixed in).

## 4. Missing PKU/XSAVE performance fix (non-security, picked up while diffing)

While diffing against the reference tree for the item above, found one more
upstream improvement this repo's tree was missing: `hardware_setup()` didn't
set `X86_FEATURE_PKU` in `kvm_cpu_cap_set()` when both XSAVE and PKU are
present on the host. Without it, a guest's `xcr0` differs from the host's,
forcing an `xcr0` switch on every VM entry/exit -- expensive, and especially
bad "when running PVM in L1" (nested), since writing `xcr0` there VM-exits to
L0. Guest-side PKU itself still isn't implemented; this only keeps the
guest's `xcr0` matching the host's to avoid the switching cost. Picked up
verbatim from `virt-pvm/linux@pvm-612`.

## What was checked and found *not* to be the cause

- **The documented `spectre_v2=off` / RSB-VMEXIT `int3` host crash** described
  in this project's own write-up is real, but is specifically tied to a
  *nested Bochs-BIOS* CPU's non-architectural RSB-underflow behavior
  colliding with the standard `FILL_RETURN_BUFFER` mitigation -- it is not a
  PVM-specific bug (PVM's `MITIGATION_ENTER` macro reuses the exact same
  generic `X86_FEATURE_RSB_VMEXIT`/`RSB_VMEXIT_LITE` infrastructure every
  vanilla VMX/SVM KVM host already uses on every VM-exit). Research turned up
  no recent upstream fix or generalization of this specific interaction, and
  no evidence it reproduces on non-nested hardware. Treat the boot-cmdline
  workaround as real but host-specific, not a general-fleet root cause --
  and see the operational recommendation below regardless.
- **`pvm_vcpu_run`/`pvm_vcpu_run_noinstr`** (the guest-entry hot path) matches
  vanilla KVM's own `noinstr`/`guest_state_enter_irqoff()` conventions closely
  and shows no obvious irq/preempt-discipline bug on inspection. The one
  self-documented risk already in the tree -- the `MC_VECTOR` TODO about
  `kvm_machine_check()` potentially running schedule-capable code in an
  irqoff context -- is real but gated behind an actual hardware machine
  check, not something normal operation would trigger.
- **The 6.12->7.1 forward-port patch** (`patches/pvm6.12to7.1complete.patch`)
  applies cleanly (`git apply --check`, exit 0) against a pristine
  `torvalds/linux` v7.1 checkout, with no conflict markers and only
  pre-existing, already-known TODOs (e.g. "TODO: handle the second part for
  #VC") carried forward unchanged. It targets a base this repo hasn't yet
  moved to, so it isn't implicated in the fleet's current symptoms, but it is
  a genuinely usable artifact if/when this project does that forward-port.

## Operational recommendation: stop relying on tribal-knowledge boot flags

The required host kernel cmdline (`pti=off fred=off spectre_v2=off` on a host
lacking FSGSBASE/RDTSCP) exists only as prose in this skill. If it is applied
per-host by hand, a reimaged/reprovisioned host silently reverting to a
default cmdline reproduces exactly the "sometimes" pattern reported. Capture
the required cmdline as a versioned, applied-by-provisioning artifact (a GRUB
config drop-in or equivalent), and add a boot-time self-check in
`hardware_cap_check()` that loudly warns (or refuses to load) when PVM detects
it is running on a host where the dangerous combination is present but the
mitigating cmdline isn't -- turning a silent future crash into an actionable,
loud diagnostic at module-load time.
