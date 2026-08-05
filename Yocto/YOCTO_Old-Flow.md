
`local.conf` : configures almost all aspect of build system. 
`bblayers.conf` : layers information configuration. 

`BB_NUMBER_THREADS` 
- number of task bitbake will perform in parallel. 
- command : `bitbake -e core-image-minimal | grep BB_NUMBER_THREADS
`PARALLEL_MAKE`
* Corresponds to -j make option. 
* specifies number of processes GNU can run in parallel. 
```bash 
# custom configuration 
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j8"
# make sure you add -j

# make sure its = " <number> "
```

> `bitbake -e ` : will give info about all environment variables. 

> generally, everything in `local.conf` should be moved to your distro configuration. 

## Building yocto image for beaglebone-yocto. 

```bash 
# source poky build environment. 
# will also make build_bbb folder as build. 
source poky/oe-init-build-env build_bbb

# change machine to beaglebone-yocto. 
uncomment the beaglebond-yocto. 

# start the build. 
bitbake core-image-minimal 


# after completing the build. the images will be saved in folder 
<build_folder>/tmp/deploy/images/beaglebone
```

After the build, images will be ready at `<build_folder>/tmp/deploy/images/beaglebone`
Image Folder will contain 
1. first-level bootloader MLO
2. second-level bootloader `u-boot`
3. kernel image 
4. device tree blob files
5. root filesystem archive 
6. module archive (kind of bundle or zip)

for Dependency issue, `./poky/scripts/install-buildtools` 

--- 
## creating partition on sdcard and copying the image stuff

```bash 

# unmount the partition. 
# unmount all paritiion. 
unmount /dev/sdb1 

# launch fdisk utility and delete the previous partitions. 
sudo fdisk /dev/sdb1
sudo fdisk /dev/sdb2

# create parition of BOOT. type primary. 
# partition 1 -> 32 GB 
# partiion 2 -> ROOTFS remaining all the space.

# creates 2 partition. 

a # creates or makes first partition bootable. 
# will enable bootable flag for parition 1. 
L # choose type of format. : FAT32 LBA. 

# 2nd partition type : 83 (linux), ext2 type. 


# first partition format as FAT 
sudo mkfs.vfat -n "BOOT" /dev/sdb1
# second parition as ext4 filesystem 
sudo mkfs.ext4 -L "ROOT" /dev/sdb2

```


copying image to SD CARD
BOOT partition 
1. MLO
2. u-boot.img
3. zImage (kernel)
4. am335x-boneblack.dtb 

ROOT partition  : uncompress core-image-tar.bz2 to the parition. 

```bash 
sudo tar -xf core-image-minimal-beaglebone-yocto.tar.bz2 -C /media/$USER/ROOT/

```

## Adding `meta-ti` layer to bbone-black

`meta-ti` BSP layer for Texas Instrument Hardware. 

steps 
```bash 

# clone the `meta-ti` source code.
# goto meta-ti -> README, see what this layer depends on. 

# in layer folder : ./conf/machine 
# the conf listed files, are all individual hardware supported by meta-ti layer. 


# make sure we are on same branch as our poky. 
git checkout scrathgap 


#        ADDING  LAYER 

#        MANUAL APPROACH 
## include the location of meta-ti layer in bblayer.conf file.  <- manual approach
# checking with command 
bitbake-layers show-layers 


#        ADDING LAYER USING BITBAKE command 
bitbake-layers add-layer <path-to-layer>

# here 
bitbake-layers add-layer ../layers/meta-arm/meta-arm ../layers/meta-ti/meta-ti-bsp/ ../layers/meta-ti/meta-beagle/ ../layers/meta-arm/meta-arm-toolchain/

# verify
bitbake-layers show-layers



#     BUILD IMAGE 

# generate minimal image 
bitbake core-image-minimal 


```

###  why would we need `meta-ti` layer when `meta-yocto-bsp` already there ?

meta-yocto-bsp would provide basic functionality. the minimum, generic hardware capabilities from beagle-bone-black. To unlock full potential, full hardware using capabilities, we need to use layer configuration provided by manufacturer itself, ti instruments. meta-yocto-bsp would provide basic cpu functionalities, kernel support, but lets say hardware has image processing unit, which is propreitery to texas instrument. that chip support, or board module support would be provided in hardware specification defined in meta-ti layer. 




# Recipes 

