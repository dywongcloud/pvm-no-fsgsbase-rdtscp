# Provisioning artifacts for PVM hosts lacking FSGSBASE/RDTSCP

The host kernel cmdline this project's hosts need (`pti=off fred=off
spectre_v2=off`) used to exist only as prose in
`skills/references/runtime-fixes.md`. Applied per-host by hand, a
reimaged/reprovisioned host silently reverts to a default cmdline and
reproduces exactly the "host sometimes freezes/resets" pattern that
prompted the 2026-08 stability investigation. This directory versions the
cmdline as an artifact your provisioning should install, so the
configuration survives reimaging.

## What's here

- `grub.d/99-pvm-host.cfg` — a drop-in for Debian/Ubuntu-style GRUB
  (`/etc/default/grub.d/`) appending the required flags to
  `GRUB_CMDLINE_LINUX`. That variable (rather than
  `GRUB_CMDLINE_LINUX_DEFAULT`) is deliberate: it applies to every
  generated menu entry, *including recovery mode*, so a recovery boot —
  say, while debugging the very instability these flags exist to prevent —
  does not silently come up with default mitigations.

## Install (Debian/Ubuntu)

```
sudo install -m 0644 provisioning/grub.d/99-pvm-host.cfg /etc/default/grub.d/
sudo update-grub
sudo reboot
```

For distributions without `/etc/default/grub.d/` support, append the same
three flags to `GRUB_CMDLINE_LINUX` in `/etc/default/grub` (or the
equivalent for your bootloader) through your configuration-management tool
— the point is that the flags live in a versioned, automatically applied
artifact, not in shell history.

## Verify after reboot

```
grep -o 'pti=off\|fred=off\|spectre_v2=off' /proc/cmdline   # expect all three
```

The `/proc/cmdline` check above is the authoritative drift test — do not
substitute "the module loaded quietly" for it. `kvm-pvm` self-checks at
load time, but each check keys on the dangerous *state*, not on the
presence of the flags: it refuses to load when KPTI or FRED is actually
active (a missing `pti=off` is silent on Meltdown-unaffected CPUs, and a
missing `fred=off` is always silent because FRED defaults off), and —
since the 2026-08 series — `hardware_cap_check()` in
`kernel/arch/x86/kvm/pvm/pvm.c` logs a prominent warning when the host is
a hypervisor guest masking FSGSBASE and RDTSCP while RSB-fill-on-VM-exit
is still active. `spectre_v2=off` is one way those RSB-fill feature bits
go quiet (and the recommended one for this host class), but not the only
way — eIBRS-class configurations can also leave them clear — so a silent
module load does not prove the drop-in survived reimaging.

## Why each flag

- `pti=off` — the PVM switcher cannot run under kernel page-table
  isolation; `kvm-pvm` refuses to load with KPTI enabled.
- `fred=off` — host FRED entry/exit is not supported by the switcher;
  `kvm-pvm` refuses to load with FRED enabled.
- `spectre_v2=off` — on a host of this class (an outer hypervisor masking
  FSGSBASE/RDTSCP from CPUID), the standard RSB-fill-on-VM-exit
  mitigation sequence was observed to collide with the outer CPU's
  non-architectural RSB-underflow behavior and crash the host with a
  spurious `int3`. That observation is host-specific — it is documented
  against a nested Bochs-BIOS outer CPU, and there is no evidence it
  reproduces on non-nested hardware (see the caveats in
  `skills/references/runtime-fixes.md`) — but `spectre_v2=off` is the
  known-good configuration on the verified host, applied fleet-wide to
  these dedicated hosts as a deliberate operational choice. Disabling
  spectre_v2 mitigations clears the `RSB_VMEXIT`/`RSB_VMEXIT_LITE`
  feature bits and NOPs that sequence out.

## Security tradeoff — read before applying

These flags disable the host's Meltdown (KPTI) and Spectre-v2 mitigations.
That is an explicit tradeoff this project makes to run PVM on this host
class. Apply the drop-in **only** to dedicated PVM hosts, never to
general-purpose machines, and treat guest workloads accordingly (the same
tenant-isolation caveats as any mitigations-off hypervisor host).
