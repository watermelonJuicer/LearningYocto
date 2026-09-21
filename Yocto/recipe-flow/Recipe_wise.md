# `makedevs`

bbfile 
```bash 
SUMMARY = "Tool for creating device nodes"
DESCRIPTION = "${SUMMARY}"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://makedevs.c;beginline=2;endline=2;md5=c3817b10013a30076c68a90e40a55570"
SECTION = "base"
SRC_URI = "file://makedevs.c"

S = "${WORKDIR}"

FILES:${PN}:append:class-nativesdk = " ${datadir}"

do_compile() {
	${CC} ${CFLAGS} ${LDFLAGS} -o ${S}/makedevs ${S}/makedevs.c
}

do_install() {
	install -d ${D}${base_sbindir}
	install -m 0755 ${S}/makedevs ${D}${base_sbindir}/makedevs
}

do_install:append:class-nativesdk() {
	install -d ${D}${datadir}
	install -m 644 ${COREBASE}/meta/files/device_table-minimal.txt ${D}${datadir}/
}

BBCLASSEXTEND = "native nativesdk"

```
`do_fetch` is not present, so `base_do_fetch` will be called. (printed in logs. )
The task that fetches pre-requisities  is `do_prepare_recipe_sysroot`. 

## how it figures out dependencies for compilation of makedevs?

**`DEPENDS` in the recipe. bb file if we added anything as DEPENDS**
When we write `DEPENDS = "zlib openssl"`, BitBake turns it into task-level edges: `do_prepare_recipe_sysroot` of our recipe depends on `do_populate_sysroot` of zlib and of openssl. 

`makedevs` doesnt have `DEPENDS` in .bb file. 
Neither have `do_configure` override. 

the depends are cross-compilation pre-requisites.  Fixed by base.bbclass function. 

default depends 
```bash
BASE_DEFAULT_DEPS = "virtual/${HOST_PREFIX}gcc virtual/${HOST_PREFIX}compilerlibs virtual/libc"
DEPENDS:prepend = "${BASEDEPENDS} "
```

## what `do_configure` step does ???


