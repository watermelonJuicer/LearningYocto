Open-Embedded Core layer, defines main classes and metadata, which standalone builds distro-less functional Linux image. 
Open-Embedded core layer, on top of bitbake, defines classes and function which would build functional linux image that can be bootable on QEMU (zero hardware support image). 

> Real Hardware support requires BSP layer (which defines stuff related to real hardware), sitting beside OE-core layer. {exactly the reason why meta-ti and meta-beaglebone exists as seperate layer}

Builds and defines on top of bitbake, 
	-> bitbake `do_Compile` = compile wherever there is makefile. 
	-> oe-core `do_Compile` = bitbake do_compile + extra stuff. 
	-> defines packaging subsystem `package.bbclass` that bitbake had no concept of it at all. 

# Major elements of oe-core layer 

## Cross-Compilation/ sysroot infrastructure 

`do_populate_sysroot`, `native.bbclass`, `nativesdk.bbclass`, `cross.bbclass`, `crosssdk.bbclass`
solely provided by oe-core. setting up cross-compiler, that compiles on source machine for target architecture. 

A compiler running on our host machine, can only compile for host machine architecture. What if we want to cross-compile. 
We cant install generic toolchain, as solution for first problem but that generic toolchain, is already built against someone's specific choice of libraries, headers etc. Someone else's choice of C library version, kernel headers version, float ABI, and feature set.

Yocto build wants a specific, reproducible combination of all libraries — matched exactly to our `MACHINE`/`DISTRO` config. So OE-Core doesn't trust an external compiler at all: **it builds its own cross-compiler from source, as part of the build itself**, using the exact same recipe/task machinery as everything else.

Why sysroot?
Once we have Standard cross-compiler, suppose we want to cross-compile some software, but where to look for headers and so file. cant look at standard path, as it would contain out native host headers and so files. 
**The sysroot is the fix: a directory tree, sitting on out build host, shaped exactly like a target root filesystem** (`/usr/include`, `/usr/lib`, `/usr/bin`) — but populated with _ARM_ headers and _ARM_ libraries. The compiler is pointed at this fake root instead of host's real one. It behaves completely normally; it just believes it's living on a different machine.

building the target C library needs a working cross-compiler, but a "complete" cross-compiler traditionally wants a target C library to link its own support code against. chicken and egg problem. So, we have staged build for our cross-compiler. 

binutils-cross (assembler, linker for target libc) -> `gcc-cross minimal cross-compiler` -> glibc (target libc built using minimal cross compiler in stage 2) -> gcc-cross fully functional. 

Every Recipe in later stage, consumes this corss compiler or output of this part. 

## Task Implementations

`base.bbclass, autotools.bbclass, cmake.bbclass, meson.bbclass, kernel.bbclass, module.bbclass`. Build specific classes (`autotools.bbclass`, `cmake.bbclass`)

Gives `do_fetch` -> `do_compile` -> `do_install` real content. 

### WHAT ?
BitBake's engine gives us task _names_ and a graph that runs them in order — that's it. 
The engine has no opinion on what `do_compile` should actually _do_. 
Something has to supply the real content: the actual sequence of commands that turns raw source code into a compiled, staged, installable artifact. General framework of compiling, packing and installing. 
Given source code and a target architecture, what are the concrete, repeatable steps to build and install it?
### WHY ?

**Avoid rewriting Boiler plate code again and again.** 
Almost every **autotools-based project** builds the same way: `./configure`, `make`, `make install`. If every one of OE-Core's hundreds of **autotools-based recipes** hand-wrote that sequence, you'd have near-identical shell code copy-pasted everywhere — a maintenance and correctness nightmare.

**Guarantee cross-compilation correctness, centrally.**

**Give every downstream mechanism one uniform interface — this is the deepest reason.** Packaging, sstate signatures, image construction — none of them care whether a recipe used autotools, cmake, or a hand-rolled Makefile. They only need to know: "did `do_compile` finish, did `do_install` produce a `D`." 
Task implementation standardizes the _interface_ (`do_configure`/`do_compile`/`do_install` always mean the same thing, always produce `S`→`B`→`D`) so every other OE-Core pillar can treat any recipe uniformly, regardless of what's actually inside.

### HOW ?
```md
SRC_URI  (recipe just states WHERE source lives)
    │
    ▼
 do_fetch   ── truly generic — same for every recipe (base.bbclass)
    │            downloads via BitBake's own fetcher library
    ▼
 do_unpack  ── truly generic (base.bbclass)
    │            extracts into WORKDIR → produces S
    ▼
 do_patch   ── truly generic (base.bbclass)
    │            applies patches, regardless of what's being built
    ▼
 ═══════════ from here, behavior forks by build system ═══════════
    │
    ▼
 do_configure ── EMPTY in base.bbclass
    │              filled in by: autotools.bbclass / cmake.bbclass /
    │              kernel.bbclass — each knows ITS project type's
    │              real configure step
    ▼
 do_compile   ── EMPTY in base.bbclass, filled in per build system
    ▼
 do_install   ── EMPTY in base.bbclass, filled in per build system
    │              always ends by staging into DESTDIR = D
    ▼
    D   ── handoff into Packaging (the pillar you already have)
```

The pipeline's job is producing three directories every later pillar depends on:
- **`S`** — unpacked, patched source (output of fetch→unpack→patch)
- **`B`** — where compilation actually happens (same as `S` for autotools by default; a separate out-of-tree dir for cmake)
- **`D`** — a fake root filesystem (`do_install`'s `DESTDIR`) containing exactly what this one recipe would install onto a real target

**`D` is the handoff point to the next pillar.** Packaging picks up exactly where task implementation stops — it never touches `S`/`B` again, only walks `D`.

## Packaging system

`package.bbclass`, `package_ipk/_rpm/_deb.bbclass`
`do_package`, `do_packagedata`, `do_package_write_*`. The entire "compiled output → installable, splittable package" concept — `PACKAGES`/`FILES` splitting logic lives here.

### WHAT ?
`do_install` has produced `D` — one **undifferentiated** directory tree containing _everything_ a recipe could possibly install: the binaries, headers, static libraries, debug symbols, locale files, documentation — all mixed together, no distinctions made yet.
**Packaging is the process of slicing that one blob or one big group of file into multiple, independently-installable named units** — each with its own file list and its own runtime dependency list. 

### WHY ?
Board can have limited flash memory with zero use for build-only artifacts like debug symbols, static libraries etc. 

If the whole `D` blob got dumped onto every image, we would ship megabytes of dead weight on every device. Splitting lets `IMAGE_INSTALL` cherry-pick only the runtime slice.

Once packaged, a recipe's output sits in a shared **package feed** — reusable across `core-image-minimal`, `core-image-full-cmdline`, and eventually your own BeagleBone image, with zero recompilation. 

A binary linked against `libssl.so.3` will crash at startup if that library isn't present on the target. With hundreds of recipes producing thousands of binaries, no one can hand-track "which binary needs which shared library, and which package provides it."
The packaging system solves this by inspecting **each binary's actual linked libraries** and auto-injecting the correct `RDEPENDS` — **this is the mechanism that makes `do_rootfs`'s "pure offline assembler" model actually work.** Without automatic, correct RDEPENDS, `do_rootfs` would have nothing reliable to walk.

### HOW ???
```
do_install output: D
      (one undifferentiated tree — everything
        this recipe could ever produce)
                       │
                       ▼
                  do_package
      ┌───────────────────────────────────┐
      │ slice D per-recipe into named      │
      │ sub-packages (main, -dev, -dbg...) │
      │ scan binaries for shared-lib links │
      │   → auto-derive RDEPENDS           │
      └───────────────────────────────────┘
                       │
             ┌─────────┼──────────┐
             ▼         ▼          ▼
          pkg-A     pkg-A-dev   pkg-A-dbg
             │
             ▼
            do_packagedata
   (publishes pkg-A's name/version/RDEPENDS
    into a SHARED, cross-recipe database —
    this is what lets a DIFFERENT recipe's
    do_package answer "who provides libX.so?")
             │
             ▼
     do_package_write_ipk  (or _deb / _rpm)
             │
             ▼
          package feed
    (arch-sorted, on-disk — exactly what
     do_rootfs later consumes)
```
## Image Construction 

defines `image.bbclass`, `rootfs*.bbclass`, `wic`
### WHAT ?

It turns a pile of independently-installable packages from packaging system into one bootable filesystem (linux file system), and then into actual file format which can be flashed on device. 
### WHY ?

**A package feed alone isn't a bootable thing.** Hundreds of `.ipk` files sitting in `DEPLOY_DIR` don't boot on a BeagleBone. Something has to pick the _right subset_, resolve everything that subset transitively needs, and lay it out as an actual root filesystem with `/etc`, `/usr`, `/lib` in the right places.

**"Filesystem directory" and "file you can flash" are two different things.** A populated rootfs directory on your Ubuntu disk isn't yet an `.img` file, a `.wic` multi-partition disk image, or a `tar.gz`. Something has to convert the assembled directory into the concrete container format your deployment mechanism (SD card, network boot, OTA update) actually needs.

### HOW ?
```
IMAGE_INSTALL + IMAGE_FEATURES   (seed list — "what I want")
                    │
                    ▼
         expand IMAGE_FEATURES → real packages
         (via FEATURE_PACKAGES_<feature> — you already know this)
                    │
                    ▼
      walk RDEPENDS closure against the package feed
      (feed = exactly what Packaging's do_packagedata/
       do_package_write_* pillar produced)
                    │
                    ▼
               do_rootfs
      target package manager runs OFFLINE, pointed at
      the feed — unpacks archives + runs postinstall
      scriptlets into a rootfs directory on your host
                    │
                    ▼
          rootfs post-processing
   (build ld.so.cache, reconcile /etc/passwd, strip
    packaging metadata if "package-management" feature
    wasn't requested, apply read-only-rootfs tweaks...)
                    │
                    ▼
               do_image_*
    convert the populated rootfs directory into actual
    container format(s) per IMAGE_FSTYPES:
    ext4 / tar.gz / cpio(initramfs) / wic(multi-partition)
                    │
                    ▼
             do_image_complete
                    │
                    ▼
              DEPLOY_DIR_IMAGE
        (the file you'd actually flash to an SD card)
```

### Software recipes 

`recipes-core`, `recipes-devtools`, `recipes-kernel` etc. 
The actual bulk content — hundreds of `.bb` files: busybox, glibc, systemd/sysvinit, util-linux, opkg/rpm/dpkg, e2fsprogs, the `linux-yocto` kernel recipe skeleton, `quilt-native`, gcc/binutils sources.

extra configuration it provides : `default/base configuration`, `QA/sanity infrastructure`

