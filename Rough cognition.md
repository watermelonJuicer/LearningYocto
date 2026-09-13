
Resolving the core-image-minimal 

local.conf 

MACHINE ??= "qemux86-64"
DISTRO ?= "poky"
EXTRA_IMAGE_FEATURES ?= "debug-tweaks"

# Reading and understaning Yocto projects 

Path
- **Tree 1 (config/environment):** `bblayers.conf` → each layer's `conf/layer.conf` → `local.conf`'s `MACHINE`/`DISTRO` → the matching `conf/machine/*.conf` and `conf/distro/*.conf`. This tree answers "what's _visible_, and what are the global knobs set to."
- **Tree 2 (recipe/class):** the target name (`core-image-minimal`) → its `.bb` file → its `inherit` chain → the `.bbclass` files those pull in → whatever _they_ `inherit`/`require`/`include`. This tree answers "what does _this specific target_ actually do."

## path 1 
`LAYERDEPENDS` : depends on which layer 


