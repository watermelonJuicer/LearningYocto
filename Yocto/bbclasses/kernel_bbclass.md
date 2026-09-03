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
- The patched, unpacked source of linux source (whatever version etc selected in kernel recipe) lives here. 
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



----
# COGNITION 

## Steps of kernel.bbclass mapped. 


do_patch : untouched
	same as whats defined in do_patch by patching.bbclass. 

kernel_do_configure : Full override of `do_configure`
	Replaces `oe-core` `base_do_configure.
	TBD 

kernel_do_compile : Fully overrides `do_compile` by base.bbclass 
	clears host flags, sets up native pkg-config env, computes reproducible `KBUILD_BUILD_TIMESTAMP`
	loops `oe_runmake` once per `KERNEL_IMAGETYPE_FOR_MAKE` entry
	handles the "alternate initrd" second pass.

do_transform_kernel : new Task Added. 
	`addtask transform_kernel after do_compile before do_install`
	post-processes image types. 

do_shared_workdir : new Task 
	`addtask shared_workdir after do_compile before do_compile_kernelmodules`

do_compile_kernelmodules : New Task 
	`addtask compile_kernelmodules after do_compile before do_strip`
	runs `make modules`, copies `Module.symvers` back to `STAGING_KERNEL_BUILDDIR` for external modules' symbol resolution.

do_kernel_link_images : New Task 
	`addtask kernel_link_images after do_compile before do_strip`
	sits in parallel with `do_compile_kernelmodules`, both hanging off to `do_compile`

do_strip : New Task 
	`addtask strip before do_sizecheck after do_kernel_link_images`
	strips the `vmlinux` ELF image specifically (not modules, not other packages), with optional `KERNEL_IMAGE_STRIP_EXTRA_SECTIONS` handling.

do_sizecheck : New Task 
	`addtask sizecheck before do_install after do_strip`
	fails/warns if the built image(s) exceed `KERNEL_IMAGE_MAXSIZE`.
	matters for boards with fixed-size boot partitions.

kernel_do_install  : override do_install task. 
	`base_do_install` is empty task, so it fills it up. 

do_bundle_initramfs : New Task 
	`addtask bundle_initramfs after do_install before do_deploy`
	optional second kbuild compile pass baking an initramfs cpio into the image, only if `INITRAMFS_IMAGE` + `INITRAMFS_IMAGE_BUNDLE="1"`.

do_package : not override, but defines variables. 

do_populate_sysroot : 

kernel_do_deploy : override of empty skeleton function
	`kernel_do_deploy`, further extended by `do_deploy[prefuncs] += "read_subpackage_metadata"` and `kernel-devicetree.bbclass`'s `do_deploy:append()`. **Role:** copies the final image(s)/module tarball/initramfs-bundled image into `DEPLOYDIR`, handling the "latest" symlink naming.