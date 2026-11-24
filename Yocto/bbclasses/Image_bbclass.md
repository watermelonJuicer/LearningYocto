# QUESTIONS or Thinking flow 

1_ : the main framework that bitbake provides by default. `base.bbclass` providing functions and whats the flow of the functions. Need to search on that and understand better. 
2_ : If `base,bbclass` provides set of functions to all, then what `image.bbclass` provides on top of it, and how it provides functionality on top of it. 
3_ : Trace image.bbclass flow and find how it executed what it does. 
4_ : bitbake understanding on how it resolves the DAG. 

Now, bbclass or bitbake can be views as function of `bbclass` files. So mainFramework is bitbake, which needs to be decoded with hello world. 



---
# Cognition
## Major variables in `bbclass` for flow. 

Each variable, answers one of 5 questions or can be said that its functions belongs to one of the 5 bins. 
1. what goes in ? what goes into image compilation? The input. The package to include. 
2. What Format images comes out? what are the format we want image in ?
3. How Big ? how big is image allowed to be ?
4. Custom Steps in Flow ? Do we need to add extra custom Steps in the flow ?
5. What is it called? Associated with naming of the final image.

### What goes in ?

`IMAGE_INSTALL` : list of name of all the package names, that has to be included in the image. 

```bash
# in core-image.bbclass
IMAGE_INSTALL ?= "${CORE_IMAGE_BASE_INSTALL}"

# in core-image-minimal.bbclass
IMAGE_INSTALL = "packagegroup-core-boot ${CORE_IMAGE_EXTRA_INSTALL}"
# keep in mind above is hard assignment. 

# WHENEVER we want to add extra features
# we add list in ${CORE_IMAGE_EXTRA_INSTALL}. 
# We don't touch IMAGE_INSTALL variable. 
```

`IMAGE_FEATURES` or `EXTRA_IMAGE_FEATURES`

Predefined image features, which can be added or removed like how we want. 
we always edit  : `EXTRA_IMAGE_FEATURES`

### OUTPUT format 

`IMAGE_FSTYPES` : list of image types to be generated. 
each entry becomes, `do_image_<fstype>` task. 

`EXTRA_IMAGECMD:<fstype>` : Extra arguments for the mkfs tool. 
example : `EXTRA_IMAGECMD:ext4 = "-i 4096 -L bbroot"`

### Size 
`set_image_size` function runs before every `do_image_<type>` function. 

| Variable                   | Default                  | Meaning                                                       |
| -------------------------- | ------------------------ | ------------------------------------------------------------- |
| `IMAGE_ROOTFS_SIZE`        | `??= "65536"` (64 MiB)   | a **floor**, not a target                                     |
| `IMAGE_OVERHEAD_FACTOR`    | `?= "1.3"`               | headroom multiplier on the real content                       |
| `IMAGE_ROOTFS_EXTRA_SPACE` | `?= "0"`                 | fixed KiB added on top. **This is the one you usually want.** |
| `IMAGE_ROOTFS_ALIGNMENT`   | (recalled: `1`)          | rounding unit in KiB                                          |
| `IMAGE_ROOTFS_MAXSIZE`     | unset                    | build fails if exceeded                                       |
| `INITRAMFS_MAXSIZE`        | `??= "131072"` (128 MiB) | same check for initramfs images                               |
### Naming and deploy location 
- `IMAGE_BASENAME ?= "${PN}""`. Defaults to the recipe name. [raw.githubusercontent.com](https://raw.githubusercontent.com/openembedded/openembedded-core/scarthgap/meta/classes-recipe/image-artifact-names.bbclass)
- `IMAGE_MACHINE_SUFFIX ??= "-${MACHINE}""` [raw.githubusercontent.com](https://raw.githubusercontent.com/openembedded/openembedded-core/scarthgap/meta/classes-recipe/image-artifact-names.bbclass)
- `IMAGE_NAME_SUFFIX ??= ".rootfs""` [raw.githubusercontent.com](https://raw.githubusercontent.com/openembedded/openembedded-core/scarthgap/meta/classes-recipe/image-artifact-names.bbclass)
- `IMAGE_LINK_NAME ?= "${IMAGE_BASENAME}${IMAGE_MACHINE_SUFFIX}${IMAGE_NAME_SUFFIX}""` [raw.githubusercontent.com](https://raw.githubusercontent.com/openembedded/openembedded-core/scarthgap/meta/classes-recipe/image-artifact-names.bbclass)
- `IMAGE_VERSION_SUFFIX ?= "-${DATETIME}""` [raw.githubusercontent.com](https://raw.githubusercontent.com/openembedded/openembedded-core/scarthgap/meta/classes-recipe/image-artifact-names.bbclass)
- `IMAGE_NAME ?= "${IMAGE_LINK_NAME}${IMAGE_VERSION_SUFFIX}""` [raw.githubusercontent.com](https://raw.githubusercontent.com/openembedded/openembedded-core/scarthgap/meta/classes-recipe/image-artifact-names.bbclass)
- From bitbake.conf: `DEPLOY_DIR_IMAGE ?= "${DEPLOY_DIR}/images/${MACHINE}"`



## FLOW with `core-image-minimal`

core-image-minimal.bb
```bash
IMAGE_INSTALL = "packagegroup-core-boot ${CORE_IMAGE_EXTRA_INSTALL}"
inherit core-image

IMAGE_ROOTFS_SIZE ?= "8192"

# if systemd in DISTRO_FEATURES, then add 4096 value otherwise not. 
IMAGE_ROOTFS_EXTRA_SPACE:append = "${@bb.utils.contains("DISTRO_FEATURES", "systemd", " + 4096", "", d)}"
```

core-image.bbclass
```bash

CORE_IMAGE_BASE_INSTALL = '\
    packagegroup-core-boot \
    packagegroup-base-extended \
    \
    ${CORE_IMAGE_EXTRA_INSTALL} \
    '
CORE_IMAGE_EXTRA_INSTALL ?= ""
IMAGE_INSTALL ?= "${CORE_IMAGE_BASE_INSTALL}"

inherit image
```












