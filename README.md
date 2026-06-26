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
