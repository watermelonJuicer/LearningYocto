This is the only one of the three that's _not_ Yocto-owned — it's OE's, mirrored into Poky. It's also the only one that's mandatory in any conceivable sense — ==remove it and nothing parses==.

BitBake is what executes tasks. `meta` is what defines what those tasks _do_. 

"task implementation" was one of OE-Core's five pillars. meta's `classes/` directory (`base.bbclass, patch.bbclass, autotools.bbclass...`) supplies the generic, reusable implementation of do_patch, do_compile, do_configure

# Classes directories

`.bbclass` files can be included in two ways. Some `.bbclass` or `.bb` files inherit them explicitly, or we include the .bbclass file globally, so every bbfile and other files inherit them. 

There are 3 classes directories, providing 3 types of bbclassses files. 

`meta/classes` : inherit individually explicitly. `.bb` & `.bbclass` files do `inherit X` 
`meta/classes-global`:  all `.bbclass` files are inherited globally. 
`meta/classes-recipe` :  never meant to be inherited directly at all. optional extra. 

## `classes-global`

high-yield to _know about_, low-touch day-to-day — we rarely edit these, we mostly get affected by them.

The policy in bbclass in `classes-global` are to be applied uniformaly to all bbfiles. "every package passes these QA checks," "every artifact gets a license manifest".

A policy an individual recipe could silently dodge if it were opt-in per-recipe would be a useless policy. So it's switched on centrally.

Major Files to check 
`base.bbclass`, `package.bbclass`, `package_ipk_deb/rpm`, `staging.bbclass & sstate.bbclass`, `insane.bbclass`, `license.bbclass`, `buildstats` etc. 



## `classes-recipe`

recipe opts to inherit these classes. 
This is where `inherit core-image` from our `core-image-minimal.bb` deep-dive actually resolves — `image.bbclass` and the `IMGCLASSES` chain live here, because only image recipes want that behavior.

**Major files — confirmed / high-confidence:**

- `devicetree.bbclass` — confirmed classes-recipe.
- Kernel family: `kernel.bbclass`, `kernel-yocto.bbclass`, `kernel-devicetree.bbclass`, `kernel-fitimage.bbclass`, `kernel-uboot.bbclass`, `kernel-uimage.bbclass`, `kernel-arch.bbclass`, `kernel-module-split.bbclass` — all switched on via `KERNEL_CLASSES`, which the migration guide names explicitly as a recipe-level, not global, mechanism. This is your assignment territory: kernel config fragments, BBB device tree. 
- `module.bbclass`, `module-base.bbclass` — out-of-tree kernel modules as a recipe, i.e. your existing kernel-module experience wrapped in bitbake.
- `image.bbclass`, `core-image.bbclass`, `image_types.bbclass` — the chain you already mapped.
- `autotools.bbclass`, `cmake.bbclass`, `meson.bbclass` — generic upstream-software recipes.
- `allarch.bbclass` — arch-independent packages.
- `uboot-config.bbclass`, `uboot-sign.bbclass` — bootloader recipe, directly on your wic/partitioning horizon.