Kernel.bbclass provides a framework for building kernel, devicetree and modules, packing it into package, so we get the final deployable image.
OE provides us framework for flow of task, `fetch -> unpack -> patch -> configure -> compile -> package`. 
kernel.bbclass takes the oe framework and builds linux kernel using that framework.
To build kernel, it introduces some functions in between the original framework. 
It inherits from base.bbclass, modifies it so we can build linux kernel. 

**OLD OE FLOW** : 
	`fetch → patch → configure → compile → install → package → deploy`

**kernel.bbclass FLOW**
- `do_fetch`
- `do_unpack`
- `do_symlink_kernsrc`  {New TASK }
- `do_patch
- `kernel_do_configure` (do_configure)
- `kernel_do_compile` (do_compile)
- `do_transform_kernel` (new task)
- `do_shared_workdir` (new task)
- `do_compile_kernelmodules`  (new task)
- `do_kernel_link_images`
- `do_strip`
- `do_sizecheck`
- `kernel_do_install` (`do_install`)
- `do_bundle_initramfs` 
- `do_package/package_data`
- `do_populate_sysroot`
- `kernel_do_deploy` (= `do_deploy`)

Important function defined in kernel.bbclass 
```
kernel_do_configure
kernel_do_compile
kernel_do_transform_kernel
do_compile_kernelmodules
kernel_do_install
do_symlink_kernsrc
kernel_do_deploy
do_strip
do_savedefconfig
do_kernel_version_sanity_check
split_kernel_packages
emit_depmod_pkgdata
```

# Kernel building without OE framework vs kernel building with OE framework.

### kernel build flow without OE framework or manual building 

1. Get source : clone from git or tarball etc. 
2. set up the environment 
	1. export the variable values. 
	2. export `ARCH` and `CROSS_COMPILE` to point to cross-compile toolchain we installed seperately. 
3. Configure & tweak menuconfig
4. build kernel image  
	1. `make -j$(nproc) zImage`
5. Build device tree 
6. Build modules 
	1. `make modules`
7. Install modules 
	1. `make modules_install INSTALL_MOD_PATH= <path to target rootfs`
8. Deploy Image 
	1. copy `zImage` and `dtb` file in FAT parition. 

### Kernel build flow with OE

same make target step, but each one is bitbake task and is automatically executed. 
1. ARCH/CROSS-COMPILE 
	1. `kernel-arch.bbclass` maps `TARGET_ARCH` to kbuild's `ARCH`.
	2. `KERNEL_CC`/`KERNEL_LD`/etc. point at the toolchain **OE already built** for our machine.
	3. we never do EXPORT manually. 
2. defconfig/menuconfig
	1. `kernel_do_configure`, seeded either from a plain `${WORKDIR}/defconfig`.
3. `make zImage`
	1. `kernel_do_compile`
	2. wrapped with reproducible `KBUILD_BUILD_TIMESTAMP` and dependency tracking against exactly this sysroot's toolchain.
4. build device-tree : `make dtbs`
	1. `kernel-devicetree.bbclass`'s `do_compile:append`, driven by `KERNEL_DEVICETREE` in `machine.conf`.
5. build module and install `make modules` & `modules install`
	1. `do_compile_kernelmodules` / `kernel_do_install`, output landing in `${D}` where `kernel-module-split.bbclass` slices it into individually versioned `kernel-module-<name>` packages instead of a flat rsync.
6. Manual boot partition copy 
	1. `kernel_do_deploy` drops everything into `DEPLOY_DIR_IMAGE`; wic (`WKS_FILE` + `IMAGE_BOOT_FILES`) is the automated version of "copy zImage+dtb to the FAT partition."


# Flow of `kernel.bbclass`

## `do_fetch`

Untouched by kernel.bbclass. Same defination as defined in base.bbclass. 

## `do_unpack`

Does Task flag extention of do_unpack of oe-core function 

- `do_unpack[cleandirs] += " ${S} ${STAGING_KERNEL_DIR} ${B} ${STAGING_KERNEL_BUILDDIR}"`
- its adding to cleandirs, bitbake wipes these 4 directories before running it. (For fresh build, you need to clear out whats stage and whats compiled if its stale.) Its stale or not, is found by another mechanism. 

do_unpack takes whats downloaded under `${DL_DIR}` by do_fetch step, and extracts, checkout git tree etc under `${WORKDIR}`.

extends flag of base_do_unpack 
```bash
do_unpack[cleandirs] += " ${S} ${STAGING_KERNEL_DIR} ${B} ${STAGING_KERNEL_BUILDDIR}"
do_clean[cleandirs]  += " ${S} ${STAGING_KERNEL_DIR} ${B} ${STAGING_KERNEL_BUILDDIR}"
```

### `${S} =  KERNEL_STAGING_DIR`
	default in bitbake.conf : `${TMPDIR}/work-shared/${MACHINE}/kernel-source/`

- Scoped by machine. 
- The patched, unpacked source of linux (whatever version etc selected in kernel recipe) lives here. 
- Stable, predictable path. 
- Read-Only. 
- `.config`, `vmlinux` or kernel build output are not written to this directory. 
- The source is untouched by build. 

### `${B} = ${WORKDIR}/build`

`KBUILD_OUTPUT = ${B}`
It is WORKDIR scoped. Every distinct kernel build, different version etc, get its own ${B}. 
This is where `.config`, every compiled `.o`, `vmlinux`, `System.map`, `Module.symvers` actually land.

`do_unpack[cleandirs]` and `do_clean[cleandirs]` wipe this every time, because it's _entirely_ regenerable from `${S}` + a config, nothing here is precious.

### `STAGING_KERNER_BUILDDIR` : shared publish point

value : `${TMPDIR}/work-shared/${MACHINE}/kernel-build-atrifacts`
Use : 
	copy the _essential subset_ of `${B}`'s private output (`System.map`, `.config`, `Module.symvers`, generated headers
	 the kernel version/localversion strings, signing keys) into this stable, shared location.
	 The contents of this directory will be used by other recipe, for its build etc. 


## `do_symlink_kernsrc` 

New Task added by kernel.bbclass. 

Mainly does
	`addtask symlink_kernsrc before do_patch do_configure after do_unpack`
	Relocates the source unpacked from `${S}=${WORKDIR}/tree/` to `STAGING_KERNEL_DIR`

### What it is Fundamentally ?
1: Everything (recipes, and hardcoded code in kernel building recipe) looks for kernel source related stuff in `KERNEL_SRC = ${STAGING_KERNEL_DIR}`.
	`module.bbclass` reads `KERNEL_SRC=${STAGING_KERNEL_DIR}`, `kernel-arch.bbclass` bakes `${STAGING_KERNEL_DIR}` into debug-info remapping.
2: None of them look at `${S}`. However, we do `${S} = ${STAGING_KERNEL_DIR}`. 
3: That only works if one thing is guaranteed: _the real kernel source is always reachable at `STAGING_KERNEL_DIR`, no matter what any individual recipe happened to set `${S}` to._
4: `do_symlink_kernsrc` is the task that **enforces that guarantee**.
5: entire job is: _if `S` and `STAGING_KERNEL_DIR` are not already the same place, make them resolve to the same content anyway._

### WHY ?
**Why not just set up the symlink in advance and let the fetcher unpack straight through it?**
We set simlink of `${S}` and `${STAGING_KERNEL_DIR}` and in `do_unpack` step, directly unpack into `${STAGING_KERNEL_DIR}`. 
1: "We can't just create the symlink in advance as the git fetcher can't cope with the symlink."
2: do_unpack fetcher machinery, need to unpack into a **real directory**, full stop; its internal directory-handling (checking existence, cleaning stale checkouts, moving/renaming paths) isn't written to cope with unpacking into or through a symlink. 
3: So the indirection or redirection has to be introduced as a **separate step, after** unpacking finishes — which is exactly what `do_symlink_kernelsrc` task is. Let it unpack, and then as seperate task, we copy into `${STAGING_KERNEL_DIR}`. 

### HOW ? how it works, actual data flow etc ?
```python
s = d.getVar("S")
kernsrc = d.getVar("STAGING_KERNEL_DIR")

# if S != STAGING_KERNEL_DIR
if s != kernsrc:
    bb.utils.mkdirhier(kernsrc)        # ensure the target path exists
    bb.utils.remove(kernsrc, recurse=True)   # wipe it — guarantee empty target
    if s[-1] == '/':
        s = s[:-1]                     
    # strip trailing slash (os.symlink quirk-avoidance)
```

2 Cases from here. We have EXTERNSRC (external source of kernel), local copy which we want to build from. 

case 1 : no external local source
```python
BEFORE:
  ${WORKDIR}/git/            ← real directory, actual unpacked source
  ${STAGING_KERNEL_DIR}      ← just wiped, empty

  shutil.move(s, kernsrc)     # physically relocate all real content
  os.symlink(kernsrc, s)      # leave a symlink at the OLD path, pointing to the NEW real location

AFTER:
  ${STAGING_KERNEL_DIR}/            ← real directory, now holds the actual source (moved here)
  ${WORKDIR}/git → ${STAGING_KERNEL_DIR}   ← symlink; ${S} still "works" transparently
```

content phycially moves. 

CASE 2 : external local source present. 
```python
BEFORE:
  ${S} = /home/you/my-kernel-workspace/     ← real, developer-owned, must not be touched
  ${STAGING_KERNEL_DIR}                     ← just wiped, empty

  os.symlink(s, kernsrc)      # STAGING_KERNEL_DIR itself becomes a symlink pointing AT s

AFTER:
  /home/you/my-kernel-workspace/                    ← real directory, completely untouched
  ${STAGING_KERNEL_DIR} → /home/you/my-kernel-workspace/   ← symlink
```
here, we didnt copied fulll contents. we just made symlink. so, we dont delete the original source. 

## `do_patch` 

Untouched. Same as whats defined in `patch.bbclass`

## kernel_do_configure : Full override of `do_configure`
	Replaces `oe-core` `base_do_configure.
	TBD 

### Configuration of files for kernel compilation 

`.config` is the result of parsing every scattered Kconfig declaration into one dependency graph, then applying a set of concrete answers (from a defconfig, from interactive input, or from Kconfig-declared defaults where nothing else says otherwise) and resolving that graph until every symbol has exactly one consistent value — then flattening that into one text file.

#### Kconfig

_  Kconfig is declarative language for decribing config symbols.
_  Defines configuration symbols which are consumed during the kernel build time. 
_  Defines symbols, their valid value range, defaults etc. 
_  Kconfig also includes bunde of tools, `scripts/Kconfig/*`, which are scripts to parse the Kconfig Files. 
_  Kconfig is not single file, Every Directory has its Kconfig files, with its own configuable options of its own. 
_  What it defines? 
	- modules : Yes. majorly. Drivers or plugable drivers are major count in kernel, as we can swicth them off if we dont need them. 
	- Architecture/platform selection : `ARCH_OMAP2PLUS`, SoC family choices, which cascade into which other sub-trees of Kconfig even get sourced
	- Core Kernel behaviour : `PREEMPT`/`PREEMPT_NONE`/`PREEMPT_VOLUNTARY` (preemption model), `HZ` (tick rate), `NR_CPUS`, `SMP`
	- FileSystems : `EXT4_FS`, `NFS_FS`, `TMPFS`
	- Security Framework : `SECURITY_SELINUX`, `SECCOMP`
	- Debug Instrumentation Infra : `ASAN`, `LOCKDEP`, `FTRACE`, `KGDB`
	- Memory management tunables : 
	- Networking, crypto, cgroups, power management choices, ALSA, USB STACKS

#### `defconfig`, `menuconfig`, `oldconfig`
All three are way, kconfig are parsed and configurable options in kconfigs are set value of and ultimately made or flattened into one big `.config` file which kernel will use as reference, while compiling the kernel. 

`defconfig`
	- `defconfig` file is sparse file. Contains the symbols which differs from their default values. 
	- All of kconfig is parsed and generated into single tree, defconfig is the value of option which differs form default values. Normal kernel, but the tweaks. 
	- there can be several `defconfigs` for single architeture, according to different use cases. 
	- Steps 
		- `scripts/kconfig/conf` first parses the _entire_ Kconfig tree fresh — building the full schema, same as it would for any other target.
		- reads selected `defconfig` sparse file and applies each line as an explicit answer.
		- For every symbol _not_ mentioned, it falls back to Kconfig's own declared default
		- The fully resolved result — one concrete value per symbol in the whole tree, not just the ~300 that were listed — gets written out as `.config`.
`menuconfig`
	- `menuconfig` starts from whatever `.config` already exists (your defconfig-seeded one, in this workflow) rather than from nothing.
	- parses the same Kconfig tree, but renders it as a navigable ncurses menu whose _structure_ literally mirrors the Kconfig source layout
	- On save, it re-runs the full resolution pass (same as step 3 above) so the result stays internally consistent, then overwrites `.config`
`oldconfig`
	- you already have a `.config` — Staring point. 
	- `.config` — say, generated against a 5.10 kernel source tree — and you've now checked out a newer tree (6.1) whose Kconfig files have gained symbols that simply didn't exist when your `.config` was written.
	- `oldconfig` walks the newly-parsed Kconfig tree, compares it against your existing `.config`, and prompts you **only for the symbols that are genuinely new** — every one of your thousands of prior answers is carried over untouched.

### `kernel_do_configure`

Kernel needs valid `.config` files to start the compilation. `kernel_do_configure` step, **"guarantee a valid, complete `.config` sits in `${B}` before `do_compile` runs — no matter where the raw configuration came from."**

How the recipe does it, 
```bash 

# ── Config-sourcing Tier 1: legacy .config sitting in S ──
    # Only fires if S != B (true for every kernel recipe), a .config
    # exists in S, and B doesn't have one yet.
    if [ "${S}" != "${B}" ] && [ -f "${S}/.config" ] && [ ! -f "${B}/.config" ]; then
        mv "${S}/.config" "${B}/.config"
    fi

    # ── Config-sourcing Tier 2: a plain defconfig staged via SRC_URI ──
    # "defconfig" here is NOT a kbuild-recognized filename — it's just
    # an ordinary saved copy of a .config (often produced by
    # `make savedefconfig`) that the RECIPE chooses to stage at this
    # exact WORKDIR path. kbuild has no idea this file exists; only
    # this shell code looks for it, then copies it to become the one
    # file kbuild actually cares about: ${B}/.config.
    if [ -f "${WORKDIR}/defconfig" ] && [ ! -f "${B}/.config" ]; then
        cp "${WORKDIR}/defconfig" "${B}/.config"
    fi

    # ── Final step: normalize whatever .config now exists ──
    # For linux-yocto (your case), ${B}/.config already exists here,
    # produced one task earlier by do_kernel_configme's merge_config.sh.
    # Both tiers above are skipped; this line is doing all the real work:
    # make -C ${S} O=${B} olddefconfig   (fall back to oldnoconfig on old kernels)
    ${KERNEL_CONFIG_COMMAND}
```


## kernel_do_compile : Fully overrides `do_compile` by base.bbclass 

_  clears host flags, sets up native pkg-config env, computes reproducible `KBUILD_BUILD_TIMESTAMP`
_  loops `oe_runmake` once per `KERNEL_IMAGETYPE_FOR_MAKE` entry
_  handles the "alternate initrd" second pass.

_"run `make <imagetype>` for each requested kernel image type, correctly."_ Everything before the final loop is preparing the _environment_ that single command needs to run correctly and reproducibly — none of it is the actual compile.

```bash 

kernel_do_compile() {
    ...
    for typeformake in ${KERNEL_IMAGETYPE_FOR_MAKE} ; do
        oe_runmake ${PARALLEL_MAKE} ${typeformake} ${KERNEL_EXTRA_ARGS} $use_alternate_initrd
    done
}
# iterate over `KERNEL_IMAGETYPE_FOR_MAKE` and run `make <target>` once per entry
```

#### Different Types of Files, output of `do_compile`

`vmlinux`
	- the raw, statically-linked kernel ELF binary.
	- Uncompressed, contains full debug/symbol information (unless stripped).
	- **kbuild's actual build _product_ — everything else is a transformation applied on top of it**
	- Lands at `${B}/vmlinux`, at the **top level** of the build directory
	- the direct input for every other compressed/wrapped format.

`System.map`
	- not a boot image at all
	- A plain-text address→symbol table, generated automatically by kbuild as a side effect of the final link step
	- whenever `vmlinux` is linked, you never ask for it explicitly). Lands at `${B}/System.map`
	- Its job: let tools decode raw kernel addresses (oops/panic dumps, `perf`, `depmod`) back into human-readable symbol names _for this exact build_, since addresses shift between builds/configs.

`Module.symvers`
	- only produced when `CONFIG_MODULES=y`.
	- A table of CRC/version checksums for every symbol the kernel exports to modules. 

`zImage/bzImage`
	- architecture-specific, _compressed, self-extracting_ boot images.
	- Built by `arch/<arch>/boot/Makefile` targets that take `vmlinux`, run `objcopy`/`strip` on it. 
	- `vmlinux` itself is not directly bootable on most architectures. Lands under `${KERNEL_OUTPUT_DIR}` = `arch/${ARCH}/boot/` → for BBB: `${B}/arch/arm/boot/zImage`.
`uimage`
	- a `zImage` (or `vmlinux`, depending on config) wrapped in an additional small header via the `mkimage` tool, purely so U-Boot's `bootm` command can recognize and load it.
`vmlinux.gz`
	- plain `gzip` of `vmlinux`

## do_transform_kernel : new Task Added. 

	`addtask transform_kernel after do_compile before do_install`
	post-processes image types. 
	

`KERNEL_IMAGETYPES`
	Full Plural wishlist. Every Kernel Image artifact, we want to end up with. 
		Starts with single value of `KERNEL_IMAGETYPE` = `zImage`
		Other classes and recipes Add up or append to the list. 
		
`KERNEL_IMAGETYPES` drives everything downstream to `do_compile` task. 
	- what `kernel_do_install` copies into `${D}/${KERNEL_IMAGEDEST}`
	- what `kernel_do_deploy` copies into `DEPLOY_DIR_IMAGE`
	- what `-image-<type>` sub-packages get generated (`kernel-image-uimage`, `kernel-image-vmlinux`, etc

`KERNEL_IMAGETYPE_FOR_MAKE`
	- subset of that wishlist the kernel's own Makefile can build directly via `make <target>`
	- This is what actually gets handed to `oe_runmake` in `kernel_do_compile`.
	- The substitution table that builds `KERNEL_IMAGETYPE_FOR_MAKE` from `KERNEL_IMAGETYPES` is **not centralized**. 
		- It's done piecemeal, by whichever class owns a synthetic type. 

Kernel_do_transform step, converts Image types exported from make to kernel_imagetypes. 
`do_compile` can compile kernel types listed in `KERNEL_IMAGETYPE_FOR_MAKE`. 
`do_transform` step, converts `KERNEL_IMAGETYPE_FOR_MAKE` to `KERNEL_IMAGE` . 


## do_shared_workdir : new Task 
	`addtask shared_workdir after do_compile before do_compile_kernelmodules`

FLOW : 
	`do_compile` -> `do_shared_workdir` -> `do_compile_kernelmodules`

Kbuild scaffolding for building _other things against_ this kernel later.

### Why This Step Exist ?
_ Every BitBake recipe gets its own private sandbox or gets it own private directory for dumping its built stuff. 
_ In later stage, for out-of-tree kernel module compilation, we would need built kernel artifacts, and generated headers from the kernel build, and we cant find this directory. 
	_ The build directory name keeps on changing according to latest push commit. 
	_ Everytime, there is update in kernel, we have to hand edit the dependency directory in out-of-tree kernel module `.bb` file and that would be big hassle. 
	_ Solution to this, is after kernel do_compile step. we Create a seperate directory, which contains important headers and dependency files, required by out-of-kernel modules, which will reference to this directory directly. 
```bash
STAGING_KERNEL_DIR      = tmp/work-shared/beaglebone-yocto/kernel-source
STAGING_KERNEL_BUILDDIR = tmp/work-shared/beaglebone-yocto/kernel-build-artifacts
```
`do_shared_workdir` = shared the working directory of kernel.
`do_shared_workdir` = entire job is to **publish** the select subset of `${B}` that other recipes need, into that shared location

### What it does : 
- `cd ${B}` — works inside the just-compiled kernel build tree.
- `kerneldir=${STAGING_KERNEL_BUILDDIR}` → `install -d $kerneldir` — ensures the destination exists. This isn't `${B}` or a sysroot path; it's a fixed, machine-wide location: `${TMPDIR}/work-shared/${MACHINE}/kernel-build-artifacts`.
- Copies a deliberately minimal, hand-picked set of files into it — the source comment literally says they've kept this list small on purpose rather than copying everything, to avoid unbounded "file creep":
    - `${KERNEL_PACKAGE_NAME}-abiversion` / `-localversion` — plain text files recording `KERNEL_VERSION` / `KERNEL_LOCALVERSION`, which `module-base.bbclass` reads later to know exactly which version string to target.
    - `System.map-${KERNEL_VERSION}`, `.config`, `Module.symvers` (if it exists yet)
    - `include/config/kernel.release`, `include/config/auto.conf`
    - `include/generated/*` and `arch/${ARCH}/include/generated/*` (whole directories, copied recursively)
    - Conditionally: module-signing keys (`certs/signing_key.*`), `include/linux/version.h` on older kernels, a PowerPC-specific `crtsavres.o` quirk, `tools/objtool/objtool` if `CONFIG_UNWINDER_ORC=y`, and `scripts/basic` / `scripts/gcc-plugins`

### Actual Flow 

`do_compile` output -> `{B}`
```
# Files copied to ${STAGING_KERNEL_BUILDDIR}

${B}/.config
${B}/vmlinux, ${B}/arch/arm/boot/zImage   (whatever's in KERNEL_IMAGETYPE_FOR_MAKE)
${B}/System.map
${B}/Module.symvers          ← exists but INCOMPLETE at this point
${B}/include/generated/*
${B}/include/config/kernel.release
${B}/include/config/auto.conf
${B}/certs/signing_key.*     (if signing enabled)
```
`Module.symvers` incomplete after `do_compile` step? Because it's generated from the vmlinux link — it only knows about symbols exported _for code that got linked into vmlinux_. Symbols exported specifically for loadable modules aren't finalized until modules actually get built. Hold that thought.

`do_shared_workdir` -- ouput -> `${STAGING_KERNEL_BUILDDIR}`
It copies the dependencies files. Thats all. nothing else. 

`do_compile_kernelmodules`
	_ This task builds entirely against its own recipe's private `${B}`, which it already has full access to (same recipe, same WORKDIR — no isolation boundary to cross). It doesn't need anything `do_shared_workdir` produced to do its job.
	_  `do_compile_kernelmodules`, will use `${B}` do compile kernel modules which are in-tree, and will also give out updated `module.symvers` file with updated symbol table. 
	_ `do_compile_kernelmodules` will then copy the update symbol table file back to STAGING_KERNEL_BUILDDIR. 

`do_shared_workdir` : 
	populates the `${STAGING_KERNEL_BUILDDIR}` for out of tree compilation of modules and recipes. 
	Its like saving the artifacts of build kernel to constant place, so we can reference it when kernel compilation is done, and other stuff needs it. 

## do_compile_kernelmodules : New Task 

_ `addtask compile_kernelmodules after do_compile before do_strip`
_ runs `make modules`, copies `Module.symvers` back to `STAGING_KERNEL_BUILDDIR` for external modules' symbol resolution.

### What it does ?
_ **do_compile_kernelmodules** builds every Linux driver/subsystem that the `.config` marked as a _loadable module_
_ It runs the kernel's own `make modules` target, still entirely inside the build tree. 
_ Nothing gets installed or packaged here.
_ Copies the updated symbol table, after compilation of module, back to staging directory. 

### How it does it ?
```python

do_compile_kernelmodules() {
	unset CFLAGS CPPFLAGS CXXFLAGS LDFLAGS MACHINE
	if (grep -q -i -e '^CONFIG_MODULES=y$' .config); then
		oe_runmake ${PARALLEL_MAKE} modules CC="${KERNEL_CC}" LD="${KERNEL_LD}" ${KERNEL_EXTRA_ARGS}
	else
		bbnote "no modules to compile"
	fi
}
addtask compile_kernelmodules after do_compile before do_strip
```
* Unsets Flags. 
* `grep -q -i -e '^CONFIG_MODULES=y$' .config` — the actual guard. `.config` at this point is the same file `do_kernel_configme` finalized before `do_compile` ran. This single line is the entire "is there any module compiling to do" decision
* `oe_runmake ${PARALLEL_MAKE} modules CC="${KERNEL_CC}" LD="${KERNEL_LD}" ${KERNEL_EXTRA_ARGS}` — this is the actual work. `oe_runmake` is OE's thin wrapper around `make`
	* build the `modules` kbuild target — with the cross toolchain forced via `KERNEL_CC`/`KERNEL_LD` (same override from `kernel_do_compile`) and OE's computed `-j<N>` parallelism.
* `addtask compile_kernelmodules after do_compile before do_strip`. 
* `do_shared_workdir` is the thing that actually wedges itself in between `do_compile` and this task — its _own_ addtask line says `after do_compile before do_compile_kernelmodules`. 

#### Input it takes 
1. `{B}/.config`
2. `${S}` : kernel source tree. Every driver's .c files. Required for compilation. 
3. `${B}` : Build or compiled kernel source, populated by do_compile. 
	1. built-in `.o`s, `include/generated/*`, `include/config/*`, `vmlinux`, a partial `Module.symvers` covering only built-in exports (incomplete Module.symvers)
4. `KERNEL_CC / KERNEL_LD` : cross toolchain variables. same as of kernel_do_compile step. 

`STAGING_KERNEL_DIR` / `STAGING_KERNEL_BUILDDIR`. Those exist so _other recipes_ can reach into this recipe's build state through the sysroot. This task _is_ the kernel recipe, so it just uses its own `${S}`/`${B}` directly

#### OUTPUT 
1. One `.ko` per `=m` driver, sitting **in place** inside the object tree — e.g. `${B}/drivers/net/can/spi/mcp251x.ko` — not moved anywhere else yet.
2. `${B}/Module.symvers` — extended with every module's exported symbols, on top of the built-in exports `do_compile` already wrote there.
3. `${B}/modules.order`, `${B}/modules.builtin` — kbuild bookkeeping files listing build order / which subsystems are built-in.

## do_kernel_link_images : New Task 

`addtask kernel_link_images after do_compile before do_strip`
sits in parallel with `do_compile_kernelmodules`, both hanging off to `do_compile`

- **kbuild's own output layout is inconsistent by image type**
- Architecture-native compressed formats (`zImage`, `bzImage`, `uImage`) get placed by kbuild itself under `arch/$ARCH/boot/` — that's baked into the kernel's own Makefiles.
- `vmlinux` (built for _every_ config, regardless of `KERNEL_IMAGETYPE`) always lands at the top of the build directory, never under `arch/$ARCH/boot/`
- So any piece of code — inside kernel.bbclass, in a BSP layer, in a standalone script — that generically does "look under `${KERNEL_OUTPUT_DIR}` (= `arch/${ARCH}/boot`) for whatever image type I need" will find `zImage` there, but silently fail to find `vmlinux`.

This step will just copy the vmlinux, and make it available under `arch/$ARCH/boot` folder. 

## do_strip : New Task 

`addtask strip before do_sizecheck after do_kernel_link_images`
strips the `vmlinux` ELF image specifically (not modules, not other packages), with optional `KERNEL_IMAGE_STRIP_EXTRA_SECTIONS` handling.

Some boards require that we strip extra section from linux image out of vmlinux itself. Maybe size constrains etc. So this step accomplishes it. 
`KERNEL_IMAGE_STRIP_EXTRA` is variable, which is empty by default. 
If Any section defined in this, the step would strip that section out of vmlinux image. 
	Step : vmlinux copied -> iterator over section mentioned -> strip those sections out of copyied image. 
Uses cross-compiler tool to strip the image. 

#### INPUT 
File to strip : `${B}/${KERNEL_OUTPUT_DIR}/vmlinux` → `tmp/work/.../linux-yocto/6.6.23+git/build/arch/arm/boot/vmlinux`
strip tool : `${KERNEL_STRIP}`
Which sections to remove : `${KERNEL_IMAGE_STRIP_EXTRA_SECTIONS}`

#### OUTPUT 
output written to : `${B}/${KERNEL_OUTPUT_DIR}/vmlinux.stripped` → `tmp/work/.../build/arch/arm/boot/vmlinux.stripped`.
Original `vmlinux` at the same location is left completely alone.


## do_sizecheck : New Task 
	
`addtask sizecheck before do_install after do_strip`
fails/warns if the built image(s) exceed `KERNEL_IMAGE_MAXSIZE`.
matters for boards with fixed-size boot partitions.


## kernel_do_install  : override do_install task. 
	`base_do_install` is empty task, so it fills it up. 


```bash 
BASE_WORKDIR ?= "${TMPDIR}/work"
WORKDIR = "${BASE_WORKDIR}/${MULTIMACH_TARGET_SYS}/${PN}/${PV}"
D = "${WORKDIR}/image"


# FOR BBONE
${D} = tmp/work/beaglebone_yocto-poky-linux-gnueabi/linux-yocto/6.6.23+git/image/
```

### Non-kernel recipe. 
suppose we have recipe, mytool and we compile it. 
`do_install` step will copy the output (whatever we got from `do_compile` step) -> reorganize and copy it into `{D}`.
Inside `{D}` it will be places in such way that it will be placed inside file system. So, prefix can be path, in new machine it can be pasted in. 
`{D}/usr/bin/mytool` | `{D}/use/include/mytool_headers` etc. 

### kernel context
In kernel context, we have kernel modules, which goes inside `/lib/modules/` and images in `/boot/`. 
For kernel build, `do_install` takes whatever in {B}, 
	-> copies `.ko` files in `{D}/lib/modules`
	-> copies kernel image and metadata in `{D}/boot/`

OUTPUT 
```bash
${D}/lib/modules/6.6.23-yocto-standard/drivers/.../*.ko
${D}/lib/modules/6.6.23-yocto-standard/modules.order, modules.builtin, ...
${D}/boot/zImage-6.6.23-yocto-standard
${D}/boot/System.map-6.6.23-yocto-standard
${D}/boot/config-6.6.23-yocto-standard
${D}/boot/vmlinux-6.6.23-yocto-standard
${D}/boot/Module.symvers-6.6.23-yocto-standard
```


## do_bundle_initramfs : New Task 

_ `addtask bundle_initramfs after do_install before do_deploy`
_ optional second kbuild compile pass baking an initramfs cpio into the image.
_ Only if `INITRAMFS_IMAGE` + `INITRAMFS_IMAGE_BUNDLE="1"`.

InitramFS : 
	At initial booting process, kernel does initial memory management, initalize driver baked statically into it etc. 
	Then kernel requires `rootfs` to initalize other drivers and other stuff. 
	Mounting file system, itself require drivers which are in userspace. 
	So, before mounting initial file system, it mounts a ramfs, (small filesystem, which stays in ram). 
	Kernel mounts ramfs irrespect of whatever, we have initramfs provided or not. if provided, initramfs cpio gets poured into that ramfs. 
After `do_intall` we have kernel images and kernel modules sitting in `${S}`. 

If, the variables, deciding inclusion of cpio initramfs are enabled, then in this step, 
	take each kernel type, 
	for each kernel type, take backup of image already compiled. 
	recompile or repack the kernel image with cpio initramfs file. 
	paste into `{D}`. 
OUTPUT 
```
${B}/usr/rootfs-initramfs.cpio           ← decompressed, staged for kbuild to consume
${B}/arch/arm/boot/zImage                ← restored: original, non-bundled image (unchanged)
${B}/arch/arm/boot/zImage.initramfs      ← new: kernel + initramfs, one combined file
```


----
# COGNITION 

## Steps of kernel.bbclass mapped. 



do_package : not override, but defines variables. 

do_populate_sysroot : 

kernel_do_deploy : override of empty skeleton function
	`kernel_do_deploy`, further extended by `do_deploy[prefuncs] += "read_subpackage_metadata"` and `kernel-devicetree.bbclass`'s `do_deploy:append()`. **Role:** copies the final image(s)/module tarball/initramfs-bundled image into `DEPLOYDIR`, handling the "latest" symlink naming.