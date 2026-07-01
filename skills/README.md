# pvm-forward-port — a Claude skill

A reusable Claude Code skill that encodes how to **forward-port the PVM hypervisor
([virt-pvm/linux](https://github.com/virt-pvm/linux)) from an old base onto a newer mainline Linux
release** — the incremental-hop method, the concrete 6.12→7.1 API-drift fixes, the runtime bugs, the
no-FSGSBASE/RDTSCP support, the Spectre-v2 `int3` finding, and the crash-debugging techniques.

Everything in it is drawn from the real 6.12.33 → 7.1 port (`pvm-6.12-to-7.1-complete.patch`, a
110-commit series) and the write-up *"Anatomy of a 9-hop kernel rebase: bringing PVM to Linux 7.1."*
No inferred or padded content.

## What it's for

PVM is an out-of-tree, KVM-style hypervisor that runs guests **without hardware virtualization** (a
software "switcher" + shadow MMU providing `/dev/kvm`). It's 131 commits on Linux 6.12.33, living in
the two churniest parts of the kernel — the x86 **entry** assembly and the **KVM x86** core. A
forward-port is therefore **textual conflict resolution + compile-driven API-drift repair + runtime
debugging**, not "apply a patch." This skill captures that process so a future run does it faster.

## When it activates

Claude invokes it automatically when you ask to *forward-port PVM*, *rebase PVM / kvm-pvm onto a
newer kernel*, *port the Pagetable-based Virtual Machine to `<version>`*, *continue the PVM mainline
port*, or *update kvm-pvm for Linux 7.x*.

## Install

Personal (all your projects):
```sh
mkdir -p ~/.claude/skills
tar xzf pvm-forward-port-skill.tar.gz -C ~/.claude/skills/
```
Or per-project (checked into a repo): extract into `<repo>/.claude/skills/` instead.
Claude Code auto-discovers it on the next session — no config needed.

## Layout

```
pvm-forward-port/
├── SKILL.md                              # entry point: CPU context, the core rule, procedure, landmines
├── README.md                             # this file
├── references/
│   ├── method.md                         # tree setup, per-hop rebase, the conflict taxonomy
│   ├── api-drift-catalog.md              # every 6.12→7.1 symbol/signature break + its fix
│   ├── runtime-fixes.md                  # the 2 runtime bugs + no-FSGSBASE/RDTSCP + spectre_v2 int3
│   ├── debugging.md                      # crash-survivable techniques when printk is dead
│   └── build-validate-operate.md         # config, the validation ladder, host-safety discipline
└── scripts/
    └── hop.sh                            # per-hop rebase driver (auto-resolves scaffolding, stops on PVM-core)
```

`SKILL.md` is the focused entry Claude reads first; the `references/` files are loaded on demand so
context stays lean until a specific detail is needed.

## The one rule

**Incremental, one stable release at a time, and BUILD at every hop** (`6.12 → 6.13 → … → 7.1`). A
direct big-jump rebase is lossy — take-ours/union auto-resolvers silently drop PVM additions and
duplicate others; it compiles-but-misbehaves. The compile step at each hop is the whole game: most
breakage is *silent API drift* (the patch applies but a signature moved underneath it).

## Highlights it will save you from re-discovering

- **CPU context first:** FSGSBASE/RDTSCP are ISA features on both Intel and AMD; the fallback series
  is inert on feature-complete hosts (recent EPYC/Xeon) and only needed where a nested instance masks
  them. The `int3`/RSB finding is micro-architecture-dependent — `spectre_v2=off` was the fix on the
  nested Bochs target, not a universal constant.
- **7.1 needs `CONFIG_X86_FRED=y`** even without FRED hardware (software FRED dispatch for the KVM
  EXTINT path); extend the six `CONFIG_KVM_INTEL` gates to `|| CONFIG_KVM_PVM`.
- **7.1 namespaced KVM exports** → register `kvm-pvm` as a 3rd vendor sub-module in `kvm_types.h`.
- **Two runtime bugs** a guest run exposes: MSR-hypercall routing (`pvm_{get,set}_msr`, not the common
  handler) and the host `task_pt_regs` restore (or Firecracker SIGSEGVs at a guest RIP).
- **`load_seg_legacy()` FS-base caching** silently corrupts guest TLS — force the `MSR_FS_BASE` write.
- **The `int3` host crash is a mitigation**, fixed with `spectre_v2=off` (keeps MDS/TAA/MMIO/L1TF).
- **Host-safety discipline:** keep the known-good kernel as GRUB `saved_entry` + `panic=5`, test with
  a one-time `grub2-reboot`, `dracut --force` after every module change, restore production services
  before leaving.

## Accuracy note

If a future hop contradicts a detail here, **trust the compiler and the kdump vmcore over this doc**,
then update the relevant reference file. The catalog is a lookup table for a specific hop range; the
method and debugging sections are version-agnostic.
