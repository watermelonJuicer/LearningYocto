# Introduction

## Elements of Building embedded linux 
	1. ToolChain : compiler, assembler, linker. The program or Tool which generates or compiles the image, main kernel etc. 
	2. Boot loader : the program which boots the kernel. 
	3. Kernel : main brain. the one which manages the resources of system. 
	4. filesystem : the filesystem, where all program and files resides.  

OF the above listed elements, bootloader, kernel and filesystem are what goes inside the linux image. 
ToolChain is used to build the rest of stuff. 
## Yocto 

INPUT : 
			- kernel configuration, hardware specification, packages & binaries we need to install etc in our kernel 
			- (basically the description of linux we want to install in our system)
OUTPUT : our custom Embedded linux distribution
			- (Linux Kernel, Root File system, Bootloader, Device Tree)

Yocto is standardized version for building custom linux images. 
It borrowed the framework from open-embedded, recipes and parsing and compilation engine, selected only most important recipes, the minimal recipes, and standardized hardware. 
Yocto is kind of framework, which borrows down from open-embedded but has rigirous testing, and standard support from industry. 

With yocto, we have poky. the golden reference of yocto. anyone can fork yocto and use its framework.
Yocto gave its own reference using its own framrework called poky. 

we can add various extra receipe in poky by adding new receipes, adhering to yocto framework. 
## Poky 
working reference of yocto project. 
yocto refers to system used to build our custom embedded linux project, poky is one of the working example of yocto. 
yocto at technical level is combined repo of component 
* bitbake 
* OpenEmbedded Core 
* meta-yocto-bsp
* Documentation

### metadata (in yocto terminology)
- metadata refers to instructions for build or instruction which are used at build time to build linux image (here). 
- command and data used to indicate version of software. 
- where to obtain the information or code or data for compilation 
- changes or addition to software (patches)

basically metadata is all the configuration data, and additional data which specifies what to do with main data or information.
we have code, main source code of kernel in form of c files etc. what to do with those c files, compile, link etc, we need instruction for that. thats metadata. 

In yocto, metadata comes in form of: 
- configuration files (.conf)
- Recipes (.bb & .bbappend)
- class (.bbclass)
- Include (.inc) files
### meta-yocto bsp 
1. receipe containing configuration files, related to hardware board. (device files, configuration files)
2. it contains programs and other stuff related to functioning of board or hardware. (if it has screen, drivers of screen etc)
3. by default in poky, hardware already supported : TI bbone, some intel board etc. 
4. if we want to build for our custom hardware, whose support not already given, we need to add configuration in meta-yocto bsp. 

> Historically the project grew from, and works with the OpenEmbedded Project which is the build system the project uses and shares components with.

---
# Core architecture

when we build embedded linux image for certain product (router, car ECU, camera) etc, we need, 
1. specific kernel version (for uniformity in product)
2. only the package we need (no bloat, flash storage saving)
3. everything cross-compiled (we build on x86, we need for some specific hardware)
4. consistent reproducible build. (same output everytime)

For building embedded linux for any product or hardware, we have to
1. fetch source code. 
2. cross compile for target architecture. 
3. all the compiler output components, must be packaged into single filesystem. 
4. filesystem be packaged into bootable image. 

yocto automates the above 4 steps. its not a distro, it builds or compiles distros. 

> YOCTO automates 4 steps 
> 	1. fetch
> 	2. compile
> 	3. assemble 
> 	4. package into image. 

CORE ARCHITECTURE

Layer 3 : 
	WHAT to build ?
	metadata (receipes, configs, classes)

Layer 2
	HOW to build ? 
	bitbake engine or task executor. 

Layer 1 
	WHAT is available to build?
	oe-core (base recipe library)

poky is reference project. it ships all 3 layers as starting point. 

## BITBAKE 

what  yocto does is `fetch -> compile -> assemble -> package`. 
But the software or thing which executes this steps in order is bitbake. 
It is a Task scheduler.

Bitbake 
1. reads receipes (instructions)
2. resolves dependencies between builds. 
3. executes task in order or parallel if possible. 

> bitbake order : `do_fetch → do_unpack → do_patch → do_configure → do_compile → do_install → do_package → do_rootfs → do_image`
> Each task is a shell/Python function defined in the recipe

# Metadata 
bitbake is task executor. The instructions which instructs bitbake for execution are collectively known as metadata. 
## Recipes (`.bb` files)

blueprint for building one piece of software. Contains information like 
* where is source code ? `SRC_URI`
* what version ? `PV` package version 
* what are the dependencies ? `DEPENDS`
* how to compile the software ? `do_compile`
* what to install and where ? `do_install`
```bash
# Example: a minimal recipe for "hello"
DESCRIPTION = "Hello World program"
SRC_URI = "https://example.com/hello-1.0.tar.gz"
PV = "1.0"

do_compile() {
    ${CC} hello.c -o hello
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 hello ${D}${bindir}/hello
}
```

## configuration files (`.conf` files)

Sets global variables throughout the build environment. 

| `build/conf/local.conf`      | Your personal build settings (target machine, parallel jobs, tmp dir) |
| ---------------------------- | --------------------------------------------------------------------- |
| `build/conf/bblayers.conf`   | Which layers are active in this build                                 |
| `meta-*/conf/layer.conf`     | A layer's self-description (priority, compatible Yocto versions)      |
| `meta-*/conf/machine/*.conf` | Hardware description (architecture, kernel, bootloader)               |
| `meta-*/conf/distro/*.conf`  | Distro policy (which init system, package format, libc)               |

## Classes (`.bbclass` file)

shared logic what multiple recipes inherit. 

we write `.bbclass` and inherit in `.bb` files and we dont have to inherit same common stuff to every receipe falling in group. 
```
# In a recipe
inherit cmake
# Now do_configure, do_compile etc. are automatically cmake-aware
```

# Layers ( Organizing metadata )

Layer is **directory with specific structure** that groups related metadata together. 

SPECIFIC STRUCTURE
```
meta-mylayer/
├── conf/
│   └── layer.conf                         <- layer's self-description
├── recipes-core/
│   └── myapp/
│       └── myapp_1.0.bb                   <- a recipe
├── recipes-kernel/
│   └── linux-mykernel/
│       └── linux-mykernel_5.15.bb         <- recipe
└── classes/                               <-  classes here
    └── myclass.bbclass
```

why layers ? why not dump all receipes in flat folder ?
1. organization. 
2. layers have priority .( higher priority layer override lower priority layers without modifying original)
3. layers are independently versioned, tested and published. 

## OE-Core layer (most important layer)

Bitbake is just engine or software, which will parse the relevant files, resolve dependencies and build/compile stuff.
It doesnt know what `linux` is. 
someone has to tell it
	1. what is `glibc`. And how to compile it for ARM?
	2. what is busybox, what is kernel, what are the kernel components?
	3. how to take 500 compiled packages and assemble them into bootable root filesystem image?
	4. what does `cross-compile` toolchain mean?

`openembedded-core` is foundational layer. Contains global metadata, configuration and also receipes that 
is `minimum shared knowledge` needed to build working linux system from source for `any architecture`. 

sometimes its renamed as `meta` receipe, as general naming convention for receipes is `meta-*`, and oe is special base receipe on which everything builds on. 


openembedded-core (meta/) — Key Directories
=============================================

Directory                | What it contains                                  | Why you'll touch it
--------------------------------------------------------------------------------------------------------------------------------
meta/recipes-core/       | glibc, busybox, init systems, base-files          | The non-negotiable skeleton of any Linux root filesystem
----
meta/recipes-devtools/   | gcc, binutils, the cross-compilation toolchain    | This builds the compiler that builds 
																				everything else
---
meta/recipes-kernel/     | linux-yocto recipe, kernel-related classes        | Builds your BeagleBone Black's kernel
---
meta/classes/            | .bbclass files - shared logic (base.bbclass,      | Every recipe inherits behavior from here —
                          | kernel.bbclass, image.bbclass)                    | this is the grammar, not the vocabulary
---
meta/conf/               | bitbake.conf, machine/architecture tuning data    | Where global build behavior is defined

------------------------------------------------------------------------------------------------------------

- Recipes for core Linux components: `glibc`, `busybox`, `systemd`, `bash`, `openssh`, `python3`, thousands more
- Core classes: `base.bbclass`, `kernel.bbclass`, `image.bbclass`
- Machine configs for reference hardware
- The base distro config skeleton
Yocto project uses OE-core as its base, and adds its own layers like 
* reference BSP layer
* distro-policy
* bitbake 
* documentation etc. 

> OE-Core is the 20% of layers that provides 80% of what any embedded Linux product needs. 

We rarely modify it — you layer on top of it.

## meta-yocto

meta-yocto is container, for 2 layers. 
	1. meta-yocto-bsp : contains information about hardware. describes the hardware fully. 
	2. meta-poky : contains information about the distro we will be compiling. 

### meta-yocto-bsp

OE-Core has the software components. But someone has to describe the _hardware_ — 
	what architecture, which bootloader, what kernel. That's a BSP (Board Support Package).

**meta-yocto-bsp** is Yocto's **reference BSP layer** — it contains hardware descriptions for generic reference boards:
- BeagleBone
- MPC8315E-RDB
- genericx86, genericx86-64
- EdgeRouter

### meta-poky (for poky, can be meta-poky, meta-poky-tiny)

Concretely, when you set DISTRO = "poky-tiny", BitBake reads poky-tiny.conf and that file answers exactly the kind of questions a real distro's identity is built from:

	What init system by default? (sysvinit vs systemd)
	What package format, if any? (poky-tiny famously says: none — no package manager at all, just a static rootfs)
	What's the default libc? (poky-tiny swaps glibc for the much smaller musl in some configurations)
	What DISTRO_FEATURES are enabled/disabled (largetile, ipv6, wifi, etc.)

IMPORTANT : 
	So poky and poky-tiny are like two different "distro philosophies" the same way Ubuntu and Arch are — except instead of being two separately-branded downloadable OSes, they're two .conf files sitting in the same layer, both feeding the same underlying oe-core recipes, just with different policy switches flipped. poky-tiny's entire reason to exist is "minimize footprint, strip everything non-essential" — that's its philosophy, exactly the way Arch's philosophy is "minimal, user-controlled."


# Poky (reference distribution)
yocto defines the standard. 
The Standard, APIs, which layer is included in standard distribution, Testing etc. 

poky is just an example, keeping standard yocto in mind. Its a reference distribution. 
> we start with poky and extend it with out own layers and reciepes. 

poky (project and its layers)
- bitbake : main engine 
- meta 
	- meta oe-core (main linux compilation components. )
	- we dont usually play with it. 
- meta-poky 
	- standard poky distribution stuff. 
- meta-yocto-bsp 
	- board support and hardware defining stuff 
- scripts 
- documentation. 

analogy : Android. 
android open source project (AOSP) : YOCTO 
	It defines standards and APIs and testing pipelines etc. 

ASOP standard project : pixel stock android build. 

samsung and other vendors takes these android standard, on top of it add their own layers and ship their custom OS with their custom hardware. 
their custom OS is still andoird but works in their way. 






