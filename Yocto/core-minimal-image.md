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

----


# COGNITION 

## Difference between recipe & package

Recipe : 
    Its a .bb file. 
    Its a script. 
    script to build a software unit. 
    Script defines, `do_fetch -> do_unpack -> ... -> do_compile -> do_install -> do_sysroot -> do_package` flow. 
    It creates artifacts, software and its variations, it doesn't install anything into kernel or linux image. 
    Kernel like other recipe is different recipe. Software unit like busybox and kernel are two different unit of software, recipes. 
    Individually they will create different software units. 

Package : 
    Named, installable unit of file with its metadata files (dependencies, version etc). 

Recipe_output:
    Full set of packages, recipe's do_package task produces. 
    one recipe -> many packages (main installable, debug package etc). 


-----
core-image-minimal.bb 
```bash
IMAGE_INSTALL = "packagegroup-core-boot ${CORE_IMAGE_EXTRA_INSTALL}"
inherit core-image
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

inherit image   # will inherit image.bbclass
```


`RDEPENDS` is what decides "must this recipe be built and packaged" (since images consume packages, not sysroots). For a normal library/tool recipe consumed by another recipe's compiler, `DEPENDS` is what forces the build.

----
## Tracking busybox recipe. 

core-image-minimal.bb 
```bash
IMAGE_INSTALL = "packagegroup-core-boot ${CORE_IMAGE_EXTRA_INSTALL}"
inherit core-image
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

inherit image   # will inherit image.bbclass
```
image.bbclass
```bash 
RDEPENDS += "${PACKAGE_INSTALL} ${LINGUAS_INSTALL} ${IMAGE_INSTALL_DEBUGFS}"

export PACKAGE_INSTALL ?= "${IMAGE_INSTALL} ${ROOTFS_BOOTSTRAP_INSTALL} ${FEATURE_INSTALL}"

```
packagegroup-core-boot.bb
```bash 

inherit packagegroup

RDEPENDS:${PN} = "\
    base-files \
    base-passwd \
    ${VIRTUAL-RUNTIME_base-utils} \
    ${@bb.utils.contains("DISTRO_FEATURES", "sysvinit", "${SYSVINIT_SCRIPTS}", "", d)} \
    ${@bb.utils.contains("MACHINE_FEATURES", "keyboard", "${VIRTUAL-RUNTIME_keymaps}", "", d)} \
    ${@bb.utils.contains("MACHINE_FEATURES", "efi", "${EFI_PROVIDER} kernel", "", d)} \
    netbase \
    ${VIRTUAL-RUNTIME_login_manager} \
    ${VIRTUAL-RUNTIME_init_manager} \
    ${VIRTUAL-RUNTIME_dev_manager} \
    ${VIRTUAL-RUNTIME_update-alternatives} \
    ${MACHINE_ESSENTIAL_EXTRA_RDEPENDS}"
```

```bash 
RDEPENDS:packagegroup-core-boot="    base-files     base-passwd     busybox     busybox-hwclock                     modutils-initscripts                     initscripts                                   netbase     busybox     sysvinit     udev     update-alternatives-opkg      v86d"

```
we get busybox, in its RDEPENDS. 

Resolving thorugh `busybox_*.bb` and what it does and how it compiles and gets included.
`busybox_*.bb` contains only version sepcific stuff. All the common files and stuff is in 
`busybox.inc` file. 

###  busbox.inc file  
- overrides, do_configure, do_compile, do_install, do_install_ptest functions. 
```bash 
DEPENDS += "kern-tools-native virtual/crypt"
```


Defining the extra type of packages, we need to create apart from other normal types. 

```bash

# defines extra packages to be made. 
PACKAGES =+ "${PN}-httpd ${PN}-udhcpd ${PN}-udhcpc ${PN}-syslog ${PN}-mdev ${PN}-hwclock"
# defines while file to be included in each type of package. 
FILES:${PN}-httpd = "${sysconfdir}/init.d/busybox-httpd /srv/www"
FILES:${PN}-syslog = "${sysconfdir}/init.d/syslog* ${sysconfdir}/syslog-startup.conf* ${sysconfdir}/syslog.conf* ${systemd_system_unitdir}/syslog.service ${sysconfdir}/default/busybox-syslog"
FILES:${PN}-mdev = "${sysconfdir}/init.d/mdev ${sysconfdir}/mdev.conf ${sysconfdir}/mdev/*"
FILES:${PN}-udhcpd = "${sysconfdir}/init.d/busybox-udhcpd"
FILES:${PN}-udhcpc = "${sysconfdir}/udhcpc.d ${datadir}/udhcpc"
FILES:${PN}-hwclock = "${sysconfdir}/init.d/hwclock.sh"


# terminal output on packages of busybox

$ bitbake -e busybox | grep ^PACKAGES=
PACKAGES="busybox-ptest busybox-httpd busybox-udhcpd busybox-udhcpc busybox-syslog busybox-mdev busybox-hwclock busybox-src busybox-dbg busybox-staticdev busybox-dev busybox-doc busybox-locale  busybox"
dipesh@dipesh-notebook:~/projects/yocto/poky/builds/core-minimal-image$ 

```


busybox cycle. 
```
do_fetch → do_unpack → do_patch → do_configure → do_compile → do_install
   → do_install_ptest → do_package → do_packagedata → do_package_write_rpm
   → (image side) core-image-minimal.do_rootfs pulls busybox's rpm in via the
     packagegroup-core-boot RDEPENDS chain we traced two turns ago, and rpm
     installs it into the rootfs.
```

### busbox.inc :: Functions
```bash 

# internal helper
def busybox_cfg(feature, tokens, cnf, rem):
    if type(tokens) == type(""):
        tokens = [tokens]
    rem.extend(['/^[# ]*' + token + '[ =]/d' for token in tokens])
    if feature:
        cnf.extend([token + '=y' for token in tokens])
    else:
        cnf.extend(['# ' + token + ' is not set' for token in tokens])
# if feature == true, add 'token=y' string to cnf list. 
# if feature == false, add '#token is not set' to cnf list
# cnf and rem are list. 

# Map distro features to config settings
def features_to_busybox_settings(d):
    cnf, rem = ([], [])
    busybox_cfg(bb.utils.contains('DISTRO_FEATURES', 'ipv6', True, False, d), 'CONFIG_FEATURE_IPV6', cnf, rem)
    busybox_cfg(True, 'CONFIG_LFS', cnf, rem)
    busybox_cfg(True, 'CONFIG_FDISK_SUPPORT_LARGE_DISKS', cnf, rem)
    busybox_cfg(bb.utils.contains('DISTRO_FEATURES', 'nls', True, False, d), 'CONFIG_LOCALE_SUPPORT', cnf, rem)
    busybox_cfg(bb.utils.contains('DISTRO_FEATURES', 'ipv4', True, False, d), 'CONFIG_FEATURE_IFUPDOWN_IPV4', cnf, rem)
    busybox_cfg(bb.utils.contains('DISTRO_FEATURES', 'ipv6', True, False, d), 'CONFIG_FEATURE_IFUPDOWN_IPV6', cnf, rem)
    busybox_cfg(bb.utils.contains_any('DISTRO_FEATURES', 'bluetooth wifi', True, False, d), 'CONFIG_RFKILL', cnf, rem)
    return "\n".join(cnf), "\n".join(rem)
# cnf, rem starts out as empty strings. 
# if fill up cnf/rem list for the given features. 
    

# X, Y = ${@features_to_busybox_settings(d)}
# unfortunately doesn't seem to work with bitbake, workaround:
def features_to_busybox_conf(d):
    cnf, rem = features_to_busybox_settings(d)
    return cnf
def features_to_busybox_del(d):
    cnf, rem = features_to_busybox_settings(d)
    return rem
    
    
#############################################################
configmangle = '/CONFIG_EXTRA_CFLAGS/d; \
		'
OE_FEATURES := "${@features_to_busybox_conf(d)}"
OE_DEL      := "${@features_to_busybox_del(d)}"
DO_IPv4 := "${@bb.utils.contains('DISTRO_FEATURES', 'ipv4', 1, 0, d)}"
DO_IPv6 := "${@bb.utils.contains('DISTRO_FEATURES', 'ipv6', 1, 0, d)}"

python () {
  if "${OE_DEL}":
    d.setVar('configmangle:append', "${OE_DEL}" + "\n")
  if "${OE_FEATURES}":
    d.setVar('configmangle:append',
                   "/^### DISTRO FEATURES$/a\\\n%s\n\n" %
                   ("\\n".join((d.expand("${OE_FEATURES}").split("\n")))))
  d.setVar('configmangle:append',
                 "/^### CROSS$/a\\\n%s\n" %
                  ("\\n".join(["CONFIG_EXTRA_CFLAGS=\"${CFLAGS} ${HOST_CC_ARCH}\""
                        ])
                  ))
}

# it creates a list of features which one to enable and which one to disable
# with long string. 
# we have OE_DEL as in defconfig, the features which are already set to No, those line to be deleted, and and then we append the lines whose values are 
# =Y. 
```

### busybox :: `do_package`
