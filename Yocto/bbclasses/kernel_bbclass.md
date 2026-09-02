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



----
# COGNITION 

## Steps of kernel.bbclass mapped. 

do_fetch : untouched 
	same as bitbake do_fetch, no modification or rewrite done. 

## do_unpack : 

Does Task flag extention of do_unpack of oe-core function 

- `do_unpack[cleandirs] += " ${S} ${STAGING_KERNEL_DIR} ${B} ${STAGING_KERNEL_BUILDDIR}"`
- its adding to cleandirs, bitbake wipes these 4 directories before running it. 

do_unpack takes whats downloaded under `${DL_DIR}` by do_fetch step, and extracts, checkout git tree etc under `${WORKDIR}`.

extends flag of base_do_unpack 
```bash
do_unpack[cleandirs] += " ${S} ${STAGING_KERNEL_DIR} ${B} ${STAGING_KERNEL_BUILDDIR}"
do_clean[cleandirs]  += " ${S} ${STAGING_KERNEL_DIR} ${B} ${STAGING_KERNEL_BUILDDIR}"
```


do_symlink_kernsrc : new task Added 
	`addtask symlink_kernsrc before do_patch do_configure after do_unpack`
	Relocates the source unpacked from `${S}=${WORKDIR}/tree/` to `STAGING_KERNEL_DIR`

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