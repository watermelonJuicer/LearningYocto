local.conf file contains all the configuration parameters we can change, for build. 

# `bblayer.conf` file

```
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /home/dipesh/projects/yocto/poky/meta \
  /home/dipesh/projects/yocto/poky/meta-poky \
  /home/dipesh/projects/yocto/poky/meta-yocto-bsp \
  "

```

the layers included in this file are `meta`, `meta-poky`, `meta-yocto-bsp`

**`meta` — OpenEmbedded-Core.**: `classes/`, `conf/`, `recipes-*/`. This is the only one of the three that's _not_ Yocto-owned — it's OE's, mirrored into Poky. It's also the only one that's mandatory in any conceivable sense — ==remove it and nothing parses==.

**`meta-poky` — Yocto's distro policy layer.** This is deliberately thin. Its job is to answer one question: _when `DISTRO = "poky"`, what opinions does that imply?_ Concretely it holds `conf/distro/poky.conf`, which sets things like the Poky-specific version string, default `PACKAGE_CLASSES`, splash/branding bits, and any policy tweaks Yocto wants layered on top of OE-Core's neutrality. Compare this to your `poky-tiny` — that's a _sibling_ distro config (also shipped from this same layer family) that strips `DISTRO_FEATURES` down hard.

**`meta-yocto-bsp` — Yocto's reference BSP layer.** Its job is to answer: _when no vendor BSP is loaded, what hardware can we still boot on?_  the meta-yocto-bsp layer maintains several "reference" BSPs including the ARM-based BeagleBone, and generic 32-bit/64-bit x86 machines. Concretely, its `conf/machine/` directory contains four files: `beaglebone-yocto.conf`, `genericarm64.conf`, `genericx86-64.conf`, `genericx86.conf`.