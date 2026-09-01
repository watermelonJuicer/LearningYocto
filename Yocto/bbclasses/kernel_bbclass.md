Kernel.bbclass provides a framework for building kernel, devicetree and modules, packing it into package, so we get the final deployable image.
OE provides us framework for flow of task, `fetch -> unpack -> patch -> configure -> compile -> package`. 
kernel.bbclass takes the oe framework and builds linux kernel using that framework. To build kernel, it introduces some functions in between the original framework. 
Kernel.bbclass inherits from base.bbclass, modifies it so we can build linux kernel. 

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


