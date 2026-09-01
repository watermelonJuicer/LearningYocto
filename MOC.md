
# First Principles 

| File                     | Info                                                                                    |
| ------------------------ | --------------------------------------------------------------------------------------- |
| [[OpenEmbedded Project]] | Open Embedded as standalone project. what is it without yocto.                          |
| [bitbake](bitbake.md)    | info about bitbake as standalone part. what is it without yocto.                        |
| [oe-core](oe-core.md)    | The second pillar or yocto. as layer                                                    |
| [[Yocto]]                | Yocto Project. Information can be repetitive, but it gives info how it works and stuff. |

# Core-image-minimal

| File                   | Info                                                         |
| ---------------------- | ------------------------------------------------------------ |
| [[core-minimal-image]] | Building yocto core minimal image and understanding the flow |
| [[meta layer]]         | disecting the meta layer in detail.                          |
| [[bbclass]]            | folder containing explanation of major bbclasses             |


# bbclasses (BASE)

| bbclass             | Info                                                                               | defined in     |
| ------------------- | ---------------------------------------------------------------------------------- | -------------- |
| [[base_bbclass]]    | base.bbclass information. ==MOST IMPORTANT==                                       | classes.global |
| [[staging_bbclass]] | staging.bbclass information. provides populate sysroot. step after the do_install. | classes.global |
| [[package_bbclass]] | package.bbclass information                                                        | classes.global |
|                     |                                                                                    |                |
# bbclasses (For building kernel)

| bbclass            | Info                 |
| ------------------ | -------------------- |
| [[kernel_bbclass]] | Kernel.bbclass info. |
|                    |                      |

learning kernel bbclass sequence 
1. kernel-arch & linux-kernel-base 
2. kernel.bbclass (line by line)
3. kernel-module-split
4. kernel-uimage, kernel-uboot, and kernel-artifact-names
5. kernel.devicetree.bbclass
6. kernel-yocto.bbclass
7. module.bbclass & module-base.bbclass



