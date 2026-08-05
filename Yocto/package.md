# Basic Concept

A package is a container format — files (which may include binaries, but also configs, libraries, scripts, docs) bundled together with metadata: name, version, dependency list, description. 

example : A binary is a book. A package is that book shipped in a labeled box: "Title: X. Requires you already own books Y and Z. Shelve at path /usr/bin." The box + label is what makes package managers useful — without it, you'd just have loose files with no way to track what needs what.

binary  = output of recipe after do_compile {compilation}. 
package = output of recipe after do_install() and do_package(). 

> recipe producing multiple binaries doesn't mean 1 binary = 1 package — grouping is rule-based, not automatic.

package needs metadata on top or raw compiled files, there must be steps which generates metadata. 
```
do_compile   →  produces raw binaries in the build dir (no packaging yet)
do_install   →  copies "what should ship" into ${D}, a fake root filesystem tree
do_package   →  walks ${D}, splits files into groups per FILES:<pkgname> rules,
                 attaches metadata to each group, writes out actual .ipk/.rpm/.deb files
```
Package doesn't exist until do_package runs.


# Searching for packages before building. 

packages don't exist pre-build.
They're a do_package output.
So "search for available packages before building" is actually a category error — what we are really searching for is recipes, because a recipe is the promise of packages, not packages themselves.

```bash 
bitbake-layers show-recipes # will list down all recipes from existing included layers. 
```


# Custom Recipe 

Building a recipe (`bitbake myrecipe`) runs it through do_compile → do_install → do_package and deposits the resulting packages in tmp/deploy/. That's it. Nothing about running bitbake myrecipe tells any image to include those packages in its rootfs.

To get it into the image's filesystem, one of two things must happen:
	1. We explicitly list it: IMAGE_INSTALL:append = " myrecipe" (in local.conf or the image recipe)
	2. Something already in the image declares it as a runtime dependency (`RDEPENDS`), so it gets pulled in automatically as a side effect.

# DEPENDS and RDEPENDS

Software has two completely different kinds of "needing something else":
	1. To build X, you sometimes need Y's headers, libraries, or tools already present on the host build machine.
	2. To run X, you sometimes need Y's shared libraries or files already present on the target device.

DEPENDS is about building, RDEPENDS is about running

## DEPENDS in detail. 

Compiling something often requires another package's headers/libraries to link against, 
DEPENDS exists to guarantee those are staged before our `do_configure/do_compile` runs.

in receipe of our app, in file `app.bb`
```
  DEPENDS = "zlib openssl"
```
MECHANISM 
	
1. BitBake resolves this at parse time. 
	It maps zlib → the recipe providing it → forces that recipe's do_populate_sysroot 
	task to complete before myapp:do_configure runs.
2. What `do_populate_sysroot` actually does
	copies zlib's headers, .so/.a files, pkgconfig files into a shared "sysroot" directory (tmp/sysroots-components/).
	Our recipe's compiler/linker then points at that sysroot, exactly like /usr/include and /usr/lib on a normal Linux box, but sandboxed per-build.

DEPENDS jobs completes at compile time. Nothing about DEPENDS touches `do_rootfs` stage. 

## RDEPENDS in detail. 

Compiled binary can dynamically link against a shared library that must physically exist on the target at runtime, something has to tell the rootfs-builder "install this too, or the binary will crash with a missing .so."

RDEPENDS is declared per-package, inside the recipe that produces that package.

in app.bb file 
```bash 
RDEPENDS:${PN} = "openssl bash"
```
This says: "the package myapp requires packages openssl and bash to be present at runtime."

# FLOW of image install
```
image.bb: IMAGE_INSTALL = "myapp"
   ↓ BitBake looks up myapp's recipe
   ↓ finds RDEPENDS:${PN} = "openssl bash"   ← declared in myapp.bb, NOT image.bb
   ↓ pulls in openssl, bash packages too
   ↓ recurses into openssl's own RDEPENDS, bash's own RDEPENDS...
   ↓ full closure feeds do_rootfs
```