# `makedevs`

## bbfile 
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

Compilation step = source_code to compile + build time decision. 
build time decision = 
* Which features on or off
* dependencies headers and libraries and where are they
* compiler architecture selection
* where things would be installed

compile step = dumb compilation. 
We need step before it to configure and write configuration to .config, or makefile etc. That step is `do_configure`. 

## Who sets the all the variables and data which appear as export. CC flags etc?

Every one of these variables is set inside BitBake's **configuration/metadata files**, and they're all resolved into one flat datastore **when BitBake parses the recipe** — before it schedules a single task (`do_fetch`, `do_configure`, `do_compile`, ...). There is no task that "sets" `CC`/`CFLAGS`/`PATH` at build-runtime; by the time any task's `run.do_*` script gets generated, the datastore is already final.

The parsing happens in a strict, layered order, and (respecting `?=`/`??=`/`:=` semantics) whatever gets parsed **last** wins:

```
bitbake.conf (the base formulas)
   ↓
MACHINE .conf  (+ its tune .inc → CPU-specific flags like -march=core2)
   ↓
DISTRO .conf   (+ included .inc files → e.g. security hardening flags)
   ↓
local.conf     (your build-wide overrides)
   ↓
every .bbclass the recipe `inherit`s, in order
   ↓
the recipe's own .bb file
   ↓
any matching .bbappend
```

the recipe (and `.bbappend`) is parsed **last**, anything you set there naturally overrides everything above it — no special mechanism needed. The convention, though, is to **append/prepend rather than replace outright**, because a plain `CFLAGS = "..."` in the recipe throws away all the tuning (`-march=core2`) and security flags (`-D_FORTIFY_SOURCE=2`, `-fstack-protector-strong`, ...) those earlier layers built up:

```bitbake
# in makedevs_1.0.1.bb (or a .bbappend)
CFLAGS:append = " -DEXTRA_DEBUG"
LDFLAGS:append = " -Wl,--no-undefined"
```

That's it — since `do_compile()`'s body directly interpolates `${CFLAGS}`/`${LDFLAGS}` ([makedevs_1.0.1.bb:12-14](vscode-webview://0h7jlfi31g5pb6hnskiu067j390l7m67hdhfpb45a2v1ihttb8mt/meta/recipes-devtools/makedevs/makedevs_1.0.1.bb#L12-L14)), the next `run.do_compile.*` would show your flag baked straight into the fully-expanded gcc command line, and `export CFLAGS="..."` at the top would also grow to include it.

If instead you want to reach into `makedevs` from **outside** the recipe (e.g. from `local.conf` or a distro/layer config, without touching the recipe file), use BitBake's recipe-specific override syntax:

## FLOW 
### `makedevs :: do_configure`

in `run.do_configure` file, we can see compiler configuration is being exported into variables, and also saved in file for referencing next time. 

### `makedevs :: do_compile`
```python
do_compile() {
	${CC} ${CFLAGS} ${LDFLAGS} -o ${S}/makedevs ${S}/makedevs.c
}
```
This `do_compile` executes. All the variables have been set before by previous steps. 

### `makedevs :: do_install`
```python 

do_install() {
	# -d :: creates directory and any parent directory. 
	install -d ${D}${base_sbindir}
	
	# copies one file, `${S}/makedevs`, to `${D}${base_sbindir}/makedevs`
	# explicitly setting its permission mode to `0755`
	install -m 0755 ${S}/makedevs ${D}${base_sbindir}/makedevs
}

```



```bash 
# generate simular bb file for variabts makedevs-native & nativesdk-makedevs
BBCLASSEXTEND = "native nativesdk"
```
It tells BitBake: "generate extra, independently-buildable **variant recipes** of this same `.bb` file, each with a different class automatically inherited — without writing separate recipe files." For `makedevs`, this produces three distinct recipes, all from one source file:

- `makedevs` — the normal target recipe (what you've been building this whole conversation)
- `makedevs-native` — a variant that builds and runs **on the build host**
- `nativesdk-makedevs` — a variant that builds for **an SDK's host environment**

```python 
FILES:${PN}:append:class-nativesdk = " ${datadir}"
```

This is the standard OE-core pairing: **any time a recipe installs a file into a non-default location, conditionally, for a specific variant/override, it needs a matching `FILES:${PN}:append` (same override) so packaging knows to pick it up.** The two lines only make sense together — `do_install:append:class-nativesdk` puts the file on disk under `${D}`; `FILES:${PN}:append:class-nativesdk` tells `populate_packages` "yes, that path belongs to the main package," for that same variant only.


## recipe :: `JDCR_makedevs`

----
### Including workdir for standalone `.c` files

```bash 
S="${WORKDIR}"
```
Above line is important, when source is standalone `.c` file. 
`do_unpack` unpacks tarballs into `${WORKDIR}`. that naturally gives path like `${WORKDIR}/makdev_1.1.0/`. 
in case of single file, `SRC_URI = "file://hello_world.c"`, it would copy paste in `${WORKDIR}`. 
And my default, `${S}=${WORKDIR}/${BPN}-${PV}`. so we need to change `S=${WORKDIR}` so, later for configure, if we cd, we dont get error, like directory not found. 


---

### Adding custom paths for file search 

BitBake doesn't search the recipe directory itself for `file://` entries. It only searches specific subdirectories (via `FILESPATH`): `${BPN}-${PV}` (i.e. `jdcr_makedev-1.6.9`), `${BPN}` (`jdcr_makedev`), and `files`, each optionally suffixed with override dirs like. 
Append your custom file path with 
```bash
FILESEXTRAPATHS:prepend := "${THISDIR}/path1:${THISDIR}/path2:"
# Keep the trailing `:` at the end 
# (needed so it joins cleanly with the existing `FILESEXTRAPATHS` value it's prepended onto)
# and separate each additional path with `:` in between.
```

---
### Naming convention of recipes

BitBake derives `PN`/`PV` from the filename by splitting on `_` and taking only the first two tokens. Your filename is `jdcr_makedev_1.6.9.bb`, which splits into `["jdcr", "makedev", "1.6.9"]`
Standard fix (not applying, per your instruction) would be to rename the file so only one `_` precedes the version, e.g. `jdcr-makedev_1.6.9.bb` (hyphen inside the name, underscore only before the version) — giving `PN="jdcr-makedev"`, `PV="1.6.9"` — or set `PN`/`PV` explicitly inside the recipe.

Recipe name and version seperated by `_`. 
For space in recipe name, use anything other than underscore. 

----
### To `append` & `prepend` to default recipe

**Is there a generic prepend/append that works either way?** 
No. `:prepend`/`:append` concatenate raw text into the existing function body — if the base is python and you prepend shell text (or vice versa), it's a syntax error in the resulting body. You always have to match whatever the base is.

Fastest way to check any given task without hunting through class files:

```bash
$ bitbake -e makdevs-jdcr | grep -n "^python do_fetch\|^do_fetch ("
```

`bitbake -e` dumps the fully expanded metadata, and it prints the function header verbatim (`python do_fetch ()` vs `do_fetch ()`), so you don't have to trace which `.bbclass` it came from.

Functions exported by python function should be appended and prepended via python
```python

```bitbake
python do_fetch:prepend() {
    bb.note("runs before the normal fetch")
}

python do_fetch:append() {
    bb.note("runs after the normal fetch")
}
```

for bash 
```python

do_compile:prepend() {
    ...
}

do_compile:append() {
    ...
}
```
---
### `do_install`
base `do_compile` & `do_install` step doesnt do anything by itself. in `base.bbclass`, it is defined `no_operation`. 
before do_install, `${D}` is defined, and is cleaned. 

---
### `do_package`

`do_package` doesn't just sort files — the tail end of it (`PACKAGEFUNCS`, then the actual `do_package_write_*` task if your build target went that far) needs a real packaging-tool binary to produce the output package file. **`rpm-native`** being in that list (and no `opkg-utils-native`/`dpkg-native`) tells you your `PACKAGE_CLASSES` is set to the RPM backend. Modern `rpm` is a genuinely heavy piece of software, and every one of those other natives is something _it_ (or one of its own dependencies) needs to build:

- **rpm's own runtime deps**: `popt` (option parsing), `lua` (embedded scriptlets), `sqlite3` (rpm's database backend), `libarchive` + `xz`/`zstd`/`bzip2`/`zlib`/`lzlib` (payload compression), `file` (magic/file-type detection, also used directly by packaging QA)
- **A TLS/network stack**, pulled in transitively (`curl` → `gnutls` → `nettle`, `libtasn1`, `libidn2`, `libunistring`, `libmicrohttpd`, `gnutls`, `libgcrypt`+`libgpg-error`)
- **Debug-info tooling**: `elfutils`, `dwarfsrcfiles` — used to split/package debug symbols (feeds the `-dbg` package from last time)
- **The autotools bootstrap chain**, needed because most of the above are themselves autotools projects: `autoconf`, `automake`, `autoconf-archive`, `libtool`, `m4`, `gnu-config`, `pkgconfig`, plus `cmake`+`ninja` (some of them build via CMake instead)
- **Generic build-time helpers** pulled in the same transitive way: `bison`/`flex`/`gperf`/`re2c` (parser/lexer generators), `perl`+`perlcross`+`readline`+`gdbm`+`ncurses` (perl and its own deps — lots of build/packaging scripts are Perl), `python3` (ditto for Python), `gtk-doc`/`texinfo-dummy` (doc-generation stand-ins many GNU-style `configure` scripts probe for), `e2fsprogs`/`util-linux`/`libcap`/`libcap-ng`/`acl`/`attr`/`libtirpc`/`libnsl2` (low-level system libs several of the above link against)

adding package_deb to be created too, 
Adding below to local.conf. 
```
# space important before package, otherwise will give error

PACKAGE_CLASSES:prepend = " package_deb" 
```
`DEPLOY_DIR` : here is where final rpm or deb packages land. 