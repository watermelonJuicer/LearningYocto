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


----

# `zlib`

## understanding The Recipe

zlib is a **library**: other programs link against it and call its functions (`deflate()`, `inflate()`, `gzopen()`, `crc32()` and so on). It has no user-facing program of its own. The dry run of `make install` showed this: everything goes under `lib/`, `include/`, `lib/pkgconfig/` and `share/man/`, and nothing goes into `bin/`.

**The binaries you saw during compile aren't installed.** `example`, `minigzip`, `examplesh` and `minigzipsh` are zlib's own test programs. The install rules never copy them. The only one that ships anywhere is `examplesh`, which `do_install_ptest` puts in the ptest directory (`/usr/lib/zlib/ptest/`) for on-target testing.

**Other programs use it in two stages:**

1. **Build time**, from the `zlib-dev` package. The program's source does `#include <zlib.h>` and links with `-lz`. The linker follows `libz.so` → `libz.so.1.3.1` and records the soname **`libz.so.1`** in the program's `NEEDED` entries.
2. **Run time**, from the `zlib` package. When the program starts, the dynamic loader looks up `libz.so.1` on the device and loads it. Only this file and `libz.so.1.3.1` need to be on the target; the headers, `.pc` file and `libz.so` symlink are build-only.

`libz.a` is also available for programs that want zlib copied into their own binary instead (static linking).

**In Yocto**, a recipe that uses zlib just adds `DEPENDS = "zlib"`. That makes zlib build first and puts its headers and libraries into that recipe's sysroot. In your `meta/` layer alone, 60 recipes do this, including binutils, gcc, git, libpng, libxml2, cairo, dropbear, elfutils, gdb and cmake. You don't have to add the runtime dependency yourself: `do_package` scans each binary's `NEEDED` entries, sees `libz.so.1`, and adds `RDEPENDS` on the `zlib` package automatically.

###  upto `do_fetch` step
```python 

# Introduciton  
SUMMARY = "Zlib Compression Library"
DESCRIPTION = "Zlib is a general-purpose, patent-free, lossless data compression \
library which is used by many different programs."
HOMEPAGE = "http://zlib.net/"
SECTION = "libs"
LICENSE = "Zlib"
LIC_FILES_CHKSUM = "file://zlib.h;beginline=6;endline=23;md5=5377232268e952e9ef63bc555f7aa6c0"

# The source tarball needs to be .gz as only the .gz ends up in fossils/
SRC_URI = "https://zlib.net/${BP}.tar.gz \
           file://0001-configure-Pass-LDFLAGS-to-link-tests.patch \
           file://run-ptest \
           file://CVE-2026-27171.patch \
           "
UPSTREAM_CHECK_URI = "http://zlib.net/"

# the SRC_URI given is of first SRC_URI field. 
SRC_URI[sha256sum] = "9a93b2b7dfdac77ceba5a558a580e74667dd6fede4585b91eefb60f03b72df23"

# When a new release is made the previous release is moved to fossils/, so add this
# to PREMIRRORS so it is also searched automatically.
PREMIRRORS:append = " https://zlib.net/ https://zlib.net/fossils/"
```
`SRC_URI` : list of everything `do_fetch` needs to fetch to build everything. 

according to prefix, the type of fetcher is used in backend. 
	`https` -> `wget` fetcher
	`file://` -> local file fetch fetcher. so `file://` is fetcher we are specifying. 

`PREMIRRORS`
	Pair of `match-pattern`, replacement pairs. URL rewrite table. 

```
PREMIRRORS:append = " https://zlib.net/ https://zlib.net/fossils/"   
	
	MEANS 
	
find `https://zlib.net/` → replace with `https://zlib.net/fossils/`

```

if we want to specify multiple `PREMIRRORS`
```
PREMIRRORS:append = " \
    https://zlib.net/           https://zlib.net/fossils/ \
    https://zlib.net/           http://mirror.acme.internal/sources/ \
    ftp://ftp.example.org/pub/  https://backup.example.org/archive/ \
"
```
For `SRC_URI = "https://zlib.net/zlib-1.3.1.tar.gz"`, BitBake now tries, in order:
1. `https://zlib.net/fossils/zlib-1.3.1.tar.gz`
2. `http://mirror.acme.internal/sources/zlib-1.3.1.tar.gz`
3. _(pair 3's host `ftp.example.org` doesn't match → skipped)_

do_fetch -> tar.gz downloaded and all the files mentioned in `SRC_URI` is moved (copied) to recipe folder. 


```python 

# do configure here, will create makefile. 
do_configure() {
	LDCONFIG=true ${S}/configure --prefix=${prefix} --shared --libdir=${libdir} --uname=GNU
}
do_configure[cleandirs] += "${B}"

# in backend, it will be called as make -j 16 shared. 
# make shared is command in makefile
do_compile() {
	oe_runmake shared
}

# will go as make install in backend. 
# install function is already written in makefile in backend. 
do_install() {
	oe_runmake DESTDIR=${D} install
}
```
Here, in do_configure step, we are createing a `Makefile` from `${S}/configure` step. Which will then be used by `do_compile` step. 
make always required a makefile. In makedevs we compiled by hand, through manual `CC` command, as we didn't had makefile. But whenever we use `oe_runmake` it will always look for `Makefile`. 

Running upto `do_configure` step, gives us build directory, and contents are 
```bash 
dipesh@dipesh-notebook:~/projects/yocto/poky_jdcr/builds/jdcr$ ls ./tmp/work/core2-64-poky-linux/zlib/1.3.1/build/
configure.log  Makefile  zconf.h  zlib.pc
```

`do_compile`, we think how its decided to compile this and that. its all written in `makefile`. `make shared` will execute that function given in makefile. 

`do_install`, we think how it know what to install in build. if we look into makefile, and go into `install` function, we know its already written what to write in folder. So, everything we think `make` does automatically, is already given by us, from user in form of `makefile` commands. 


### `ptest` or `package test`

#### what is ptest

For given library, we write some test program, which we compile along with software, and use that test-script to test our program. In native-PC, without cross-compiling, this is easy. We build software, compile test script for same pc, and run on the pc or cpu itself.
In case of cross-compiling, we cross-compile for other cpu architecture on native pc, build software and test too, but cant run. We need to ship or transfer both to board, to test. Ptest provides us that feasibilty, in yocto. 
**In yocto, ptest bbclass, provides us framework, for cross-compiling test, and generate seperate package (includes build and test) which we ship to board and run and test it. Seperate package, as we dont want the test package to end up in production.** 

Ptest elements or pieces of ptest 
1. Test programme
	1. the tests, cross-compiled for board architecture. 
	2. Built on PC or native machine in recipe's work directory. 
2. `run-ptest`
	1. tiny shell script that starts the test
	2. we write it and keep it next to recipe. 
3. `<name>-ptest`
	1. Seperate package after `do_package` step, holding above two
	2. installed on board at `/usr/lib/<name>/ptest/`
4. `ptest-runner`
	1. program on board, which finds every `/usr/lib/*/ptest/run-ptest` and runs it. (basically, it will run ptest for all the software who has ptests installed)
	2. Installed on board. 

For ptest tests, only two rules:
- The test prints one line per check: `PASS: <name>`, `FAIL: <name>` or `SKIP: <name>`.
- `run-ptest` exits with 0 if everything passed and non-zero if something failed.

#### `inherit ptest`

What `inherit ptest` automates for us 

1.. Creates new package ( `{PN}-ptest`)
	`{PN}-ptest` containing everything under `/usr/lib/<recipe_name>/ptest`
2.. Gives us empty hooks for us to fill in 
	`do_configure_ptest`, `do_compile_ptest`, `do_install_ptest`
	above are not tasks of bitbake. These are parts of it. the original added to flow as `do_compile_ptest_base`
3.. Adds three tasks that call your hooks at the right moment.
	In logs they appear as `do_configure_ptest_base`, `do_compile_ptest_base` and `do_install_ptest_base`:
	
**`ptest` bbclass, defines `do_configure_ptest_base`, `do_compile_ptest_base` and `do_install_ptest_base` as task and adds it to flow of building the recipe, for user it exposes `do_configure_ptest`, `do_compile_ptest`, `do_install_ptest`, for user to fill up. These are empty functions, which we as user fill up.** 

It Handles boring parts of installig: 
- copies `run-ptest` into `/usr/lib/<name>/ptest/`
- runs the upstream Makefile's `install-ptest` target, if it has one
- calls your `do_install_ptest`
- sets all file ownership to root
- removes your PC's paths (like `/home/dipesh/...`) from any Makefile copied to the board, because they mean nothing there

**do_compile_ptest**
- builds test programs that normal `do_compile` doesnt build. 
- runs after `do_compile`, so library is already built and tests can link against it. 
- uses cross-compiler so test program can run on board. 
Skip this hook entirely if the normal build already produces the tests

GENERIC FRAMEWORK 
```python 
do_compile_ptest() {
    # cwd = ${B}. Library already built. oe_runmake / ${CC} = cross toolchain.
    oe_runmake <a target that BUILDS the tests but does NOT RUN them>
}
```

**do_install_ptest**
copies **everything the tests need at run time** into `${D}${PTEST_PATH}`, which becomes `/usr/lib/<pn>/ptest/` on the board


### `oe_runmake`

#### What 
`oe_runmake` is just wrapper around `make`. It calls `make` in backend, with extra stuff. 
```bash
oe_runmake hello
```
Above command does 3 things, 
1. Adds Extra words/arguments. Recipe defines many arguments, so It Adds extra words/arguments to `make` command before calling it. 
2. Writes a note in log. 
3. Stops on failure and fail loudly. 
#### WHY 
Yocto makes call to `make` multiple times. and everytime, we cant write custom `make` command at everystep for building the recipe. so we have wrapper. which notes what command is being made, and also, we pass out extra argument to wrapper, which takes care of it. 
Benefits of passing all make commands through oe_runmake, 
1. we dont have to edit make files by hand at every step. 
2. Adding arguments laters, we just need to append to extra Variable, and it will be applied. 
3. provides general framework for all the yocto build steps. 

### `in-tree` and `out-tree` compilation 

In-tree compilation : source and build directory are same. the output of build, is dumped beside the source. `Build_dir == Source_Dir`
Out-of Tree compilation : source directory and build directory are different. `build_dir != source_dir`

Default in yocto is in-tree compilation, as it costs nothing. out-of-tree compilation requires us to maintain 2 directory and also give extra arguments to make, from where to take source and from where to dump output. 

IN-tree compilation 

In-tree needs no setup. 
The build system only has to know one location, which is why hand-written Makefiles, small projects, and most older software work this way.

Cost : 
1. source and generated files mix. 
2. One configuration per tree. Objects from an arm64 build sit exactly where an x86 build would write its own.
3. source directory must be writable. 
Why does Yocto tolerate `B = S` by default? `S` is already a private, disposable copy. It lives under `WORKDIR`, which is per recipe and per target architecture. Costs 1 and 2 are neutralized because Yocto isolates builds by copying the source, not because the build system separates its outputs. Out-of-tree gets the same isolation without the copy. 

OUT-of Tree compilation 

Out-of-tree makes B != S, and each cost flips into a benefit.
The price is that the build system must now keep track of two locations.

Every rule must say whether a file lives in S or B, and this is where the bugs come from. In-tree builds hide path mistakes, because S and B are the same place and a wrong reference still works. The bug only appears when someone first tries out-of-tree, which is why old software often "breaks" there.

Where each is used ???
- Out of tree only works if it was designed for two paths. 
- Default B=S. 
- **`autotools`, `cmake`, `meson`, `kernel`, `cargo`, `go` classes:** out-of-tree, with `B = ${WORKDIR}/build`, because those tools expect it.
- **By hand (zlib):** the recipe sets `B` itself, because the package supports out-of-tree but no class applies.

## `zlib-jdcr`

1.. `LICENSE` test is important in bb file. 