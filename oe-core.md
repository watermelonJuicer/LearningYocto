Open-Embedded Core layer, defines main classes and metadata, which standalone builds distro-less functional Linux image. 
Open-Embedded core layer, on top of bitbake, defines classes and function which would build functional linux image that can be bootable on QEMU (zero hardware support image). 

Real Hardware support requires BSP layer (which defines stuff related to real hardware), sitting beside OE-core layer. {exactly the reason why meta-ti and meta-beaglebone exists as seperate layer}

Builds and defines on top of bitbake, 
	-> bitbake do_Compile = compile wherever there is makefile. 
	-> oe-core do_Compile = bitbake do_compile + extra stuff. 
	-> defines packaging subsystem `package.bbclass` that bitbake had no concept of it at all. 

## Major elements of oe-core layer 

### Task Implementations

`base.bbclass, autotools.bbclass, cmake.bbclass, meson.bbclass, kernel.bbclass, module.bbclass`
Gives `do_fetch` -> `do_compile` -> `do_install` real content. 

### Cross-Compilation/ sysroot infrastructure 

`do_populate_sysroot`, `native.bbclass`, `nativesdk.bbclass`, `cross.bbclass`, `crosssdk.bbclass`
solely provided by oe-core. setting up cross-compiler, that compiles on source machine for target architecture. 

### Packaging system

`package.bbclass`, `package_ipk/_rpm/_deb.bbclass`
`do_package`, `do_packagedata`, `do_package_write_*`. The entire "compiled output → installable, splittable package" concept — `PACKAGES`/`FILES` splitting logic lives here.

### Image Construction 

defines `image.bbclass`, `rootfs*.bbclass`, `wic`

### Software recipes 

`recipes-core`, `recipes-devtools`, `recipes-kernel` etc. 
The actual bulk content — hundreds of `.bb` files: busybox, glibc, systemd/sysvinit, util-linux, opkg/rpm/dpkg, e2fsprogs, the `linux-yocto` kernel recipe skeleton, `quilt-native`, gcc/binutils sources.

extra configuration it provides : `default/base configuration`, `QA/sanity infrastructure`

