# Questions 



-----
Before BitBake compiles a single line of code, it does zero building.
It first reads metadata across every layer, builds one complete task graph (parse phase),
and only then starts executing tasks in dependency order (execute phase).

## Phase 1 : Graph Construction

Flow when we do `bitbake core-image-minimal`

1. Full metadata scan. 

Before touching our target recipe and executing, BitBake parses every `.bb/.bbclass/.bbappend/.conf`
visible across every layer listed in `bblayers.conf`.
Building an in-memory (and cached) database of every recipe's `PN`, `PACKAGES`, `DEPENDS`, `RDEPENDS`, etc.

Any custom layer, we add, should be listed in `bblayer.conf` in local.conf. if it doesnt exist there, it is not visible to bitbake. 

2. Reading the image recipe (reading `core-image-minimal.bb` recipe file)

3. Maps packages -> Recipes. 

Using the database from step 1, BitBake resolves hello-world → produced by hello-world.bb in your custom layer. (packagegroup-core-boot resolves to a recipe that's mostly empty — its only content is an RDEPENDS list pulling in busybox, base-files, init, etc. — a "meta-package" recipe, worth knowing exists as a pattern.)

4. Recursive RDEPENDS walk 
