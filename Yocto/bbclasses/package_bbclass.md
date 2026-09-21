package : self-contained, addressable unit of software. software with all its dependencies, a unit of software which contains everything, it requires to run, and can be run without needing to install or copy anything. 

The Point of `do_package` step is to take output of `do_install`, there is lot of stuff after do_install, and put or sort it into different bins. Files for debug, go into debug bins, files for compilation against this package, go into development bin, file for running application, goes into to_run application bin. 

Example, `libgreet` recipe. Non-kernel recipe. 
output after do install : 
```
${D}/usr/bin/greet
${D}/usr/lib/libgreet.so.1.0.0
${D}/usr/lib/libgreet.so.1   → symlink → libgreet.so.1.0.0
${D}/usr/lib/libgreet.so     → symlink → libgreet.so.1
${D}/usr/include/libgreet/greet.h
${D}/usr/lib/pkgconfig/libgreet.pc
```
There are lot of files, .so files are required mainly for running and maybe for compiling against too. Header files are not required for running, just development using libgreet it it provides library etc. 
`do_package` will do package-split, and sort the files into different categories. 

OUTPUT : `${WORKDIR}/package-split/`
```
packages-split/
├── libgreet/
│   └── usr/
│       ├── bin/greet
│       └── lib/
│           ├── libgreet.so.1.0.0
│           └── libgreet.so.1 -> libgreet.so.1.0.0
├── libgreet-dev/
│   └── usr/
│       ├── include/libgreet/greet.h
│       └── lib/
│           ├── libgreet.so -> libgreet.so.1
│           └── pkgconfig/libgreet.pc
└── libgreet-dbg/
    └── usr/
        ├── lib/debug/usr/bin/greet.debug
        ├── lib/debug/usr/lib/libgreet.so.1.0.0.debug
        └── src/debug/libgreet/1.0-r0/greet.c
```
`-dbg` contains files associated with libgreet, which contains debug symbols. 
`-dev` contains file which are purley for developmental purpose.
`libgreet` contains files for runing, binary and dynamic so its linked again. 
also they are arranged in directory structure, such that the suffix is path in real file system. 


After `do_install` step for each recipe, the software and its dependencies are installed in `{D}` directory. 
in `{D}` its unstructured files plus, `{D}` is different for every recipe. 

`{D}`
	- different for each recipe. 
	- default `{D}` = `${WORKDIR}/image`
		- `${WORKDIR}` is itself different for different recipes. 
	- per recipe `{D}` = `tmp/work/<arch>/<recipe>/<version-number>`
	- for each recipe, `{D}` act as `/`. 
	- software installed in `/usr/bin/`, then we have `/etc/` etc. 
	- `do_install` copies file in `{D}` using same relative path that it will use on real device. 

do_package figure out the dependencies of given software, only run-time dependecies (which it requires to run)

for each package 
	- figure out what shared library its dynamically linked against during compile time (dynamic linked, it will require at run time. if its static linked, wont be required)
		- if there is, add it in `RDEPENDS`
	- check if recipe itself provides some so file or not. 

if package itself provides .so file, add the so name in some central database, common global. In later, if some package require that .so file, it checks in global directory, and add it in `RDEPENDS` of that software. 

for each recipe, `do_package` will walk through `{D}` and figure out dependencies of given software, and store it into database. 
Then, package_ipk/deb runs, so its packaged into package, specified according to the distro. 

# Full Flow 
below is full flow from `do_install` to `do_package_write_deb`

example of my tool application. 
`mytool` depends on libgreet (another recipe giving `libgreet.so`) and libfetch (from `libfetch.so`)
## `do_compile`

`do_compile` step compiles the mytool binary and place it into `${B}` folder. 
`{B}` is build folder.
Unique for each recipe.

output : `{B}` compiled binary

## `do_install`

do_install step installs the binary into root file system. But instead of real rootfile system, we install into fake root file system, individual for each recipe.
installs the binary into its `${D}`. 
`{D}` : `${WORKDIR}/image`
```
${WORKDIR}/image/
└── usr/
	└── bin/
		└── mytool
```

## `do_package`
1. It split the binary according to packages or type of packages we assigned receipe to build. 
2. Figure out run-time dependencies of binary and add that information in some database. 

### Splitting files into packages. 

for given binary we can have multiple type of packages according to our use case. 
	`mytool` : binary package default debug symbols stripped out
	`mytool-dbg` : binary package with debug symbols, useful for debugging purpose. 
	`mytool-src`: binary package and source file, in case we are shaing a library. 
	`mytool-staticdev` : binary package, statically linked. 

`do_package` reads `PACKAGES` variable
```bash
PACKAGES = "${PN}-src ${PN}-dbg ${PN}-staticdev ${PN}-dev ${PN}-doc ${PN}-locale ${PN}"

```

it walks the list in order and for each package type, sweeps or looks into `{D}` and figure out files needed for package type. 

OUTPUT directory
`${PKGDEST}` = `${WORKDIR}/packages-split`. unique for each recipe. 
```
${WORKDIR}/packages-split/
	├── mytool/
		└── usr/
			└── bin/
				└── mytool ← stripped binary
	└── mytool-dbg/
		└── usr/
			├── lib/
				└── debug/
					└── usr/
						└── bin/
							└── mytool.debug ← the split-out debug symbols
			└── src/
				└── debug/
					└── mytool/
						└── 1.0-r0/
							└── mytool.c ← source copy,bundled into -dbg                                                                             (default)
```

### Figuring out run time dependency

for every elf binary, it looks into its low-level linking information and figure out what files its dynamically linked against. 
It stores the list of runtime dependency into some database (global). 
It generates metadata for run time dependencies. 

output : `${PKGDESTWORK}` = commonly `${WORKDIR}/pkgdata/`
```
${WORKDIR}/pkgdata/
	└── mytool/
		├── PN: mytool
		├── PV: 1.0
		├── RDEPENDS: libgreet libfetch
		├── FILES: /usr/bin/mytool
		└── DESCRIPTION: Fictional tool that greets you with the weather
```

## `do_packagedata`
takes per recipe private metadata, and pastes into common, shared directory. 
output : `${PKGDATA_DIR}` = `tmp/pkgdata/<machine>/`

for step 2 of `do_package`, we have metadata in individual recipe `{PKGDESTWORK}`, for `do_packagedata`, its common `PKGDATA_DIR`

## `do_package_write_deb/ipk`
input : 
	`${PKGDEST}/mytool/` : from step 1 of `do_package`
	`${PKGDATA_DIR}` : output of `do_packagedata`
The metadata record from input is reformatted into debian or ipk format, binary and dependencies are compressed into tar.gz, and then put into .deb archive file. 

output : `${DEPLOY_DIR_DEB}`, organized into per architecture subfolder. 



# Do_package from `package.bbclass`

```bash 

# Since bitbake can't determine which variables are accessed during package
# iteration, we need to list them here:
PACKAGEVARS = "FILES RDEPENDS RRECOMMENDS SUMMARY DESCRIPTION RSUGGESTS RPROVIDES RCONFLICTS PKG ALLOW_EMPTY pkg_postinst pkg_postrm pkg_postinst_ontarget INITSCRIPT_NAME INITSCRIPT_PARAMS DEBIAN_NOAUTONAME ALTERNATIVE PKGE PKGV PKGR USERADD_PARAM GROUPADD_PARAM CONFFILES SYSTEMD_SERVICE LICENSE SECTION pkg_preinst pkg_prerm RREPLACES GROUPMEMS_PARAM SYSTEMD_AUTO_ENABLE SKIP_FILEDEPS PRIVATE_LIBS PACKAGE_ADD_METADATA"

# Functions for setting up PKGD
PACKAGE_PREPROCESS_FUNCS ?= ""
# Functions which split PKGD up into separate packages
PACKAGESPLITFUNCS ?= " \
                package_do_split_locales \
                populate_packages"
# Functions which process metadata based on split packages
PACKAGEFUNCS += " \
                package_fixsymlinks \
                package_name_hook \
                package_do_filedeps \
                package_do_shlibs \
                package_do_pkgconfig \
                read_shlibdeps \
                package_depchains \
                emit_pkgdata"
```

## FLOW 
```python 

python do_package () {
	
	# packages = list of package to be built.
    packages = (d.getVar('PACKAGES') or "").split()
    
	workdir = d.getVar('WORKDIR')   # the given recipe private directory, Everything is under it. 
    outdir = d.getVar('DEPLOY_DIR') # Global deploy root. only checked for existence here.
    dest = d.getVar('D')            # where do_install output is. 
    dvar = d.getVar('PKGD')         # packaging scratch copy of {D}. 
    pn = d.getVar('PN')             # package name of the recipe. 
    
    
    # Resolve package version strings. 
    bb.build.exec_func("package_setup_pkgv", d)
    # setup variables write in packaged metadata. 
    bb.build.exec_func("package_convert_pr_autoinc", d)
}
```

After this, we have `package_prepare_pkgdata` step. 
	whats `pkgdata` : After `do_package` step, we have recipe split into different types of packages. step after `do_package` is `do_packagedata`, It copies all files and metadata of recipe (RDEPENDS, PROVIDES, shared-library provided) into common pool, shared global directory. 

`package_prepare_pkgdata` takes dependencies on which busybox depends on, and copies the files its dependencies provides from above mentioned global directory to recipe private directory, to use when doing packaging. Personal copy, as global copy may get tainted by any process running, and ensuring we get fresh copy of dependencies. 

Example flow of `package_prepare_pkgdata` for busybox and it depends on `libcryptx`
1. go through task graph, and figure out `busybox:do_package` dependency list, on which do_package of busybox is dependent on. 
2. For each dependency, figure out its manifest file. 
	1. manifest file for given recipe, contains what files, .so etc it provides and their location in common packagedata directory.  
3. from manifest file, figure out locaiton and copy files in recipe local directory. 

```python
	
	# calling package_prepare_pkgdata
    bb.build.exec_func("package_prepare_pkgdata", d)
    bb.build.exec_func("perform_packagecopy", d)

```

```python
	# PACKAGE_PREPROCESS_FUNCS ?= ""
	for f in (d.getVar('PACKAGE_PREPROCESS_FUNCS') or '').split():
		bb.build.exec_func(f, d)
		
	# Walks ${PKGD} and classify each type of file
	# for each elf file, extract debug info and also 
	# strip elf of debug info so it reduces size of original elf. 
	# pastes the debug info extracted in .debug file, which later
	# will be going into {PN}-dbg package. 
	oe.package.process_split_and_strip_files(d)
	
	# fixup permission of files. 
	oe.package.fixup_perms(d)

```

```python 
	
	# PACKAGESPLITFUNCS ?= " \
    #            package_do_split_locales \
    #            populate_packages"

	for f in (d.getVar('PACKAGESPLITFUNCS') or '').split():
        bb.build.exec_func(f, d)

'package_do_split_locals : splits package according to locale language support'
'populate_package : will walk through do_install output and will split into different packages. '
```


`do_package` takes where `do_install` left off. Splits into different packages. packages themselves are not standalone, but contain metadata file of what the are, their version, their depedency and what so file they need to run. 
this metadata gets packaged later into .deb file etc. 
in `do_rootfs` step, while installing .deb file, we go thorugh the metatdata and library its depedent upon, and install them too. 