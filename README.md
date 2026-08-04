# pvm-no-fsgsbase-rdtscp

Kernel patches that let the **Pagetable-based Virtual Machine (PVM)** hypervisor —
and therefore **Firecracker microVMs without KVM / hardware virtualization** — run
on commodity cloud instances whose vCPUs **mask `FSGSBASE` and `RDTSCP`** out of
guest CPUID.

PVM normally *requires* both features. This series works around their absence on
three fronts:

- **Host switcher** — `ALTERNATIVE`-gated `SWAPGS`+`MSR_KERNEL_GS_BASE` fallback for the GS-base switch.
- **RDTSCP** — trap-and-emulate in KVM (`em_rdtscp` + `EmulateOnUD`) + route guest user-mode `#UD` through `handle_ud`.
- **Guest kernel** — read/write guest FS base via PVM hypercall instead of `RD/WRFSBASE`.

No behaviour change on feature-complete hosts. Verified end-to-end: a full Ubuntu
microVM boots to a shell on a Tencent CVM lacking both features.

**Base:** [virt-pvm/linux](https://github.com/virt-pvm/linux) `pvm-612` (Linux 6.12.33) · **License:** GPL-2.0

# Where to read more:
```
https://dywongcloud.github.io/pvm-no-fsgsbase-rdtscp/
```

## Host-stability investigation (2026-08)

Investigated reports of hosts intermittently freezing, resetting, or going
SSH-unreachable. Found and fixed, in order of severity:

1. **`kernel/.gitignore`'s `*.s` rule silently dropped ~1340 tracked `.S`
   files** whenever git evaluated it case-insensitively (`core.ignorecase=
   true`) -- including `entry_64_switcher.S` itself. Restored every missing
   file from `virt-pvm/linux@pvm-612`, hardened the `.gitignore` with an
   explicit `!*.S`, and added `.github/workflows/gitignore-integrity.yml` to
   catch a recurrence on every push (forces `core.ignorecase=true` on the
   runner so it reproduces the bug class regardless of the runner's real
   filesystem).
2. **Patch 1 of the FSGSBASE/RDTSCP series (the switcher's own MSR-based
   GS-base fallback) was never actually applied** to `entry_64_switcher.S`
   -- invisible to git the whole time because of (1). Patches 2-4 *were*
   applied, which is worse than neither being applied: the soft-gated
   `hardware_cap_check()` lets `kvm-pvm` load on a host lacking FSGSBASE,
   and the switcher then executes `rdgsbase`/`wrgsbase` unconditionally --
   a guaranteed `#UD` on the first guest mode transition on exactly the
   hardware class this project targets. Reconstructed and applied patch 1,
   verified by a clean build and an objtool diff against the unpatched
   object (zero new warnings).
3. **CVE-2026-53359 ("Januscape")**, a disclosed CVSS 8.8 use-after-free in
   KVM's x86 shadow-MMU code (upstream-fixed 2026-07-09), was present in
   this repo's `arch/x86/kvm/mmu/mmu.c` -- confirmed by diffing directly
   against the fixed upstream commit. PVM shares this shadow-paging code
   path, so this was a real, guest-triggerable host-corruption bug and a
   strong candidate for the reported symptoms even under non-malicious
   guest workloads. Fixed by taking the corrected file from upstream.
4. Picked up one more upstream fix found the same way: a guest PKU/XSAVE
   `xcr0`-switching performance fix, harmless and already validated
   upstream.

Follow-up (2026-08-04): the write-up's operational recommendation is now
implemented — the required host cmdline (`pti=off fred=off spectre_v2=off`)
ships as a versioned GRUB drop-in under `provisioning/`, and
`hardware_cap_check()` warns loudly at module load when a hypervisor guest
masking FSGSBASE and RDTSCP still has RSB-fill-on-VM-exit active — the
state `spectre_v2=off` exists to clear (active KPTI/FRED were already hard
load refusals).

Full details, what was checked and ruled out (the documented `spectre_v2=off`
int3 workaround, the vcpu-run hot path, the not-yet-applied 6.12->7.1
forward-port patch), and an operational recommendation are in
`skills/references/runtime-fixes.md`. The build/validation recipe actually
used -- required packages, a working KVM_PVM-only config, and a fast
objtool-diff loop for entry-assembly changes -- is in
`skills/references/build-validate-operate.md`.
