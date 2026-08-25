Image is top Level receipe. (inherited from `image.bb` receipe)
It is rensponsible for creating the entire linux distribution from source. 

Other receipes, involves compiling or rather cross-compiling software, but image is receipe which will agregate everything, and package them, and give us linux distribution. 

Image receipe is receipe whose job is not to compile anything, but take already built or compiled packages from other recipes, assemble them into root file system, and package that rootfs into one or more bootable images. 

Image involves : compiler, tools, libraries, Bootloader, kernel, RootFilesystem.

Creating custom images 
	We often need to create our own Image receipe to add new packages or functionalities. 

there are 2 ways, 
	1. creating image from scratch (writing image.bb from scratch) <- TEDIOUS
	2. extend or modify the existing image receipe (preferable as we just edit part concerning us, we build on top of whats existing, instead of building from scratch)

core-image-minimal recipe : `meta/recipes-core/images/core-image-minimal.bb`
core-image-sate    recipe : `meta/recipes-sato/images/core-image-sato.bb`


# Package (package.md)[package.md]

Set of packages which can be included in any image. 

contains set of packages. (these packages are packages, which software depend upon in 
backend)

using package-group name in `IMAGE_INSTALL` variable installs all packages defined by the package group into the root file system of target image. 

## how to search for packages to install. 

how to list down available packages. 



# Image creation 

## Creating image from scratch 

Simplest way, inherit core-image bbclass. It provides set of image features that can be used over. 

```bash 
# create the custom layer, my-layer 
cd my-layer

# make folder image, and in that image.bb
mkdir image 
touch image/custom-image.bb 
```
in custom-image.bb file 
```bash 

# core-image.bbclass inherits image. so we are also inheriting image class, 
# by inheriting core-image. 
inherit core-image 

# to see package groups. 
bitbake -e custom-image | grep ^IMAGE_INSTALL=



```


## Adding package group to existing Image. 

From above image, lets say, we need to use `lsusb`. we dont have that functionality as we havent install package group that contains software like `lsusb`. So, we change the existing above image, and add package-group `usbutils` to our existing `custom-image`. 

open `custom-image.bb` file
```
IMAGE_INSTALL += "usbutils"
```
we are appending package group `usbutils` to existing package groups, so it gets installed additionally. 

check if this package group got included or not. 
`bitbake -e custom-image | grep ^IMAGE_INSTAL=`

`usbutils` should be in the list. 

build the image. 
`bitbake custom-image`

load the image in qemu, and try `lsusb` command to check.

## Reusing existing image (core image minimal) instead of building image from scratch

when one already predefined image fits our need and we just need minor adjustment to it, we build new image receipe, and inhert and then edit. 
we usually, edit that image receipe itself. 

suppose core-image-minimal works for us. 
we can create our custom image.bb

`touch custom-layer/image/custom-image.bb`

in that file, 
```bash
require receipes-core/images/core-image-minimal.bb 
IMAGE_INSTALL += "lsusb"
```

check, `bitbake -e custom-image | greo ^IMAGE_INSTALL=`
we should be able to see, lsusb

build 
`bitbake custom-image`


# Image Features

Questions : 
	1. Image.bb file from scratch understanding. 
	2. what are Image_features?
	3. whats the difference between image_install and image_features?

----

core-image-minimal tracing 
```
IMAGE_INSTALL = "packagegroup-core-boot ${CORE_IMAGE_EXTRA_INSTALL}"
inherit core-image
```

core-image
```bash 
# defined package features. 

CORE_IMAGE_BASE_INSTALL = "
	packagegroup-core-boot \
	packagegroup-base-extended \
	${CORE_IMAGE_EXTRA_INSTALL} \
	"

CORE_IMAGE_EXTRA_INSTALL ?= ""
IMAGE_INSTALL ?= "${CORE_IMAGE_BASE_INSTALL}"

```