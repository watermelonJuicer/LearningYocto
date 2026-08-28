package : self-contained, addressable unit of software. software with all its dependencies, a unit of software which contains everything, it requires to run, and can be run without needing to install or copy anything. 

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