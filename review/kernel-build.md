# Kernel build

- **Load when:** building a kernel, in-tree tools or selftests to check a
  change, setting up the .config of a new worktree, or deciding whether a
  change needs a build.
- **Parameters:** `$BUILD`
- **Depends on:** [core/safe-operations](../core/safe-operations.md),
  [core/waits-and-long-ops](../core/waits-and-long-ops.md)

## Always-on rules
- [GATE] Before testing, posting or pushing a code change: build it with
  `$BUILD`, the kernel first and then every tool and selftest it touches,
  and report every warning and error, because a change nobody built breaks
  the build for everyone who pulls it.
- Run every build stage through `$BUILD`, never another make command,
  because the .config and the build artifacts come from `$BUILD`'s
  compiler and options, and a stage built another way mixes the two.

## [GATE] Checklist
### Before testing, posting or pushing a code change
- [ ] The change was built, unless it only edits comments, documentation or
      patch descriptions.
- [ ] The kernel built with `$BUILD` and the configuration options the
      tree's code paths need ([required options](#required-options)).
- [ ] Every tool and selftest the change touches built with `$BUILD` from
      the same tree, after the kernel.
- [ ] Every warning and error of every stage was reported.

## Procedure
1. Decide whether the change needs a build. Every change does, except
   edits to comments, documentation and patch descriptions, which can't
   change what the code does.
2. Build in a working tree, never in a publication tree outside an apply
   ([core/safe-operations](../core/safe-operations.md#publication-trees)).
   An apply builds the target branch once, after `git am` and before the
   push ([maintain/apply-patches](../maintain/apply-patches.md#applying)),
   not before the apply.
3. In a fresh worktree, copy the .config from a built worktree of the same
   tree whose configuration has what the tools need, such as BTF and the
   subsystem's options. Never start from `make defconfig` when tools need
   BTF: defconfig leaves BTF off, the build makes a vmlinux without BTF,
   and the tools can't dump it.
4. After copying a .config or changing an option, run
   `$BUILD olddefconfig </dev/null`. Without it, the build stops at a
   prompt for each new option and looks hung.
5. Build the kernel with `$BUILD`, incrementally. Wait for it as
   [core/waits-and-long-ops](../core/waits-and-long-ops.md) describes.
6. After the kernel, build the tools and selftests the change touches from
   the same tree: `$BUILD -C tools/<tool>` and
   `$BUILD -C tools/testing/selftests/<subsystem>`. BPF-based tools and
   selftests generate vmlinux.h from the tree's vmlinux. Without one, their
   Makefiles fall back to the running kernel's BTF in
   /sys/kernel/btf/vmlinux without an error, and the tools build against
   the wrong kernel.
7. Report every warning and error of every stage.
8. Build clean (`$BUILD clean`, then the build) only in two cases: after an
   incremental build failed or looks wrong, and before a pull request or a
   new version of a patchset is posted, so that the published result
   doesn't depend on stale objects. A failing BPF tool build is cleaned
   before its source is investigated
   ([stale BPF tool builds](#stale-bpf-tool-builds)).

## Reference
### Required options
Build with the configuration options that the tree's code paths need. A
use site behind an option that is off compiles out, and an error in it,
such as a missing include, ships unnoticed. Which options a tree needs is
in the overlay.

### Stale BPF tool builds
Stale generated files in a BPF tool's build, such as vmlinux.h or the
skeletons, cause errors such as CO-RE type-suffix mismatches (`struct
task_struct___16`) or pointer-type errors in unrelated BPF files, as seen in
tools/sched_ext. When a BPF tool build fails, clean it
(`$BUILD -C tools/<tool> clean`) and build it again. Investigate the source
only when the clean build still fails.

## Customization
- `$BUILD` names the build command, such as a wrapper that sets the
  compiler, a compiler cache, the job count and the priority.
- The overlay lists each tree's required options and the worktree to copy a
  .config from.
