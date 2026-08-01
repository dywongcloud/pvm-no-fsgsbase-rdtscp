# Build, validate, and operate: what was actually run and proven this session

Everything below was executed for real against this repository's `kernel/`
tree (Ubuntu 24.04 container, gcc 13.3.0, binutils 2.42) -- not inferred.
`skills/README.md` promised this file as part of the skill's `references/`
layout; it did not exist in the repo, so this is the first version.

## Minimum host packages for `make x86_64_defconfig` + a KVM_PVM build

The container this was built in had `gcc`/`as`/`ld`/`make` but was missing
three packages `scripts/kconfig` and `tools/objtool` need. Without them the
build fails early and unhelpfully:

```
apt-get install -y flex bison libelf-dev
```

- `flex`/`bison`: `scripts/kconfig`'s lexer/parser generation
  (`scripts/kconfig/lexer.lex.c`) -- without them, `make x86_64_defconfig`
  fails at `LEX scripts/kconfig/lexer.lex.c` with `flex: not found`.
- `libelf-dev`: `tools/objtool` needs `<gelf.h>` -- without it, any build
  that reaches `tools/objtool` (which `CONFIG_STACK_VALIDATION`/
  `CONFIG_UNWINDER_ORC` require) fails with `fatal error: gelf.h: No such
  file or directory`.

## Minimal PVM-only config, disabling vendor KVM

```
make ARCH=x86_64 x86_64_defconfig
scripts/config --file .config -e KVM -m KVM_PVM -d KVM_INTEL -d KVM_AMD
make ARCH=x86_64 olddefconfig
```

**Do not** force `CONFIG_X86_FRED=y` on a 6.12.33 base just because the
6.12->7.1 forward-port notes say 7.1 needs it. On this base, `X86_FRED`
defaults off (not set in `x86_64_defconfig`, no `default` in its `Kconfig`
entry), and forcing it on triggers a real, reproduced-here link failure:

```
ERROR: modpost: "asm_fred_entry_from_kvm" [arch/x86/kvm/kvm-pvm.ko] undefined!
```

Cause: `arch/x86/entry/entry_64_fred.S` gates the definition/export of
`asm_fred_entry_from_kvm` on `#if IS_ENABLED(CONFIG_KVM_INTEL)`. With vendor
KVM disabled and only `CONFIG_KVM_PVM=m`, that guard never fires, so the
symbol a FRED-enabled kernel expects to call is never defined -- exactly the
"extend the six `CONFIG_KVM_INTEL` gates to `|| CONFIG_KVM_PVM`" landmine the
forward-port notes describe, just encountered here on 6.12.33 because of a
self-inflicted config choice rather than the base tree's own defaults. Leave
`X86_FRED` off for a 6.12.33 build; only chase this landmine for a real
7.1 forward-port, where it may be unavoidable.

## Fast, targeted correctness check for an entry-assembly change

Building the whole kernel just to validate one `.S` file is slow. This is
the loop that actually caught nothing-new-broken for the switcher patch in
under a minute per iteration, once `include/generated/asm-offsets.h` exists:

```
make ARCH=x86_64 -j4 arch/x86/kernel/asm-offsets.s   # generates asm-offsets.h + builds tools/objtool
make ARCH=x86_64 -j4 arch/x86/entry/entry_64_switcher.o
```

To compare objtool's opinion of a change against the pre-change baseline
(rather than just "no errors," since a `.S` entry-code change can silently
introduce a new stack-frame/ORC/retpoline warning that never surfaces as a
hard error):

```
cp arch/x86/entry/entry_64_switcher.o /tmp/after.o
git stash   # or otherwise put the .S back to its pre-change state
rm -f arch/x86/entry/entry_64_switcher.o
make ARCH=x86_64 -j4 arch/x86/entry/entry_64_switcher.o
cp arch/x86/entry/entry_64_switcher.o /tmp/before.o
git stash pop

./tools/objtool/objtool --stackval --orc --retpoline --rethunk /tmp/before.o
./tools/objtool/objtool --stackval --orc --retpoline --rethunk /tmp/after.o
```

Diff the two warning sets (not just check "after" is warning-free -- some
warnings, like `switcher_enter_guest+0xde: return with modified stack frame`
in this codebase, are pre-existing and unrelated to any given change; the
useful signal is whether your change adds anything the baseline didn't
already have).

## For a faster full build (bzImage + modules), trim the config

`x86_64_defconfig` builds a huge amount of driver/filesystem code irrelevant
to validating PVM/KVM changes. This cut wall-clock meaningfully on a 4-core
box without touching anything KVM/x86-core-relevant:

```
scripts/config --file .config \
  -d SOUND -d SND -d DRM -d FB -d USB_SUPPORT -d WIRELESS -d WLAN -d NFC \
  -d SCSI -d ATA -d MD -d INFINIBAND -d STAGING -d MEDIA_SUPPORT \
  -d CRYPTO_HW -d GPU -d VIDEO_DEV
make ARCH=x86_64 olddefconfig
make ARCH=x86_64 -j$(nproc) bzImage modules
```

## Cross-checking this repo's tree against upstream without a full clone

A full `git clone` of `virt-pvm/linux` pulls the whole history (multi-GB).
This is all that was actually needed to compare specific files/diff the
whole tree against a known commit:

```
git clone --depth 1 --branch pvm-612 --filter=blob:none --no-checkout \
  https://github.com/virt-pvm/linux.git ref
cd ref
git sparse-checkout disable   # or `init --cone` + `set <dirs>` to stay narrow
git checkout pvm-612          # partial clone lazily fetches blobs on demand
```

`diff -rq kernel ref -x .git` against the full checkout took under a minute
and was the fastest way to confirm exactly which tracked files had silently
diverged from the branch this repo derives from (see
`references/runtime-fixes.md` items 3-4) -- far faster than reviewing 100+
files by hand.

## Validating a not-yet-applied forward-port patch without adopting it

To check `patches/pvm6.12to7.1complete.patch` is still a usable artifact
(applies cleanly to a real v7.1, not bit-rotted) without actually forward-
porting this repo:

```
git init ref71 && cd ref71
git remote add origin https://github.com/torvalds/linux.git
git fetch --depth 1 origin v7.1
git checkout FETCH_HEAD -- .
git apply --check /path/to/pvm6.12to7.1complete.patch   # exit 0 == clean
```
