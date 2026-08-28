
# defines `do_fetch`

```bash

# register fetch in backend DAG. 
addtask fetch

# add the directories which needed to be loaded, 
# directories in which fetched source code will be downloaded.
# DL_DIR (download directory) is shared download cache for all 
# recipes. 
do_fetch[dirs] = "${DL_DIR}"

# get file checksums if defined in bbfile. 
# get_checksum_file_list and get_lic_checksum_file_list 
# are functions (python anonymus) defined in same file. 
do_fetch[file-checksums] = "${@bb.fetch.get_checksum_file_list(d)}"
do_fetch[file-checksums] += " ${@get_lic_checksum_file_list(d)}"

# is the list of functions to call before the task executes.
do_fetch[prefuncs] += "fetcher_hashes_dummyfunc"

# flag that will grant do_fetch network access.
do_fetch[network] = "1"

# defining of main do_fetch function, 
# python is defined, so d (central bitbake database)
# is readily available. 
python base_do_fetch() {
	
	# fetch the src_uri or "". if none, do nothing, return
    src_uri = (d.getVar('SRC_URI') or "").split()
    if not src_uri:
        return

    try:
	    # invoke the fetcher, and fetch the src_uri
	    # fetcher will access the uri and initialize
	    # so we can download according to different type of 
	    # uri, http, git etc. 
        fetcher = bb.fetch2.Fetch(src_uri, d)
        
        # download the source from uri to download directory
        fetcher.download()
    except bb.fetch2.BBFetchException as e:
	    # exception, if failed, appropriate message.
        bb.fatal("Bitbake Fetcher Error: " + repr(e))
}

# last line, this export functions. 
EXPORT_FUNCTIONS do_fetch do_unpack do_configure do_compile do_install

```

# defining `do_unpack`
```bash

# add do_unpack after do_fetch. 
# unless do_fetch is not completed, do_unpack cant run. 
addtask unpack after do_fetch

# unpack directory is WORKDIR (personal directroy for each recipe)
# contarary, for fetching we have common download directory
do_unpack[dirs] = "${WORKDIR}"


do_unpack[cleandirs] = "${@d.getVar('S') if os.path.normpath(d.getVar('S')) != os.path.normpath(d.getVar('WORKDIR')) else os.path.join('${S}', 'patches')}"

# defining the function
python base_do_unpack() {
    src_uri = (d.getVar('SRC_URI') or "").split()
    if not src_uri:
        return

    try:
	    # fetch, initialization
        fetcher = bb.fetch2.Fetch(src_uri, d)
        
        # unpack the downloaded stuff to WORKDIR
        fetcher.unpack(d.getVar('WORKDIR'))
    except bb.fetch2.BBFetchException as e:
        bb.fatal("Bitbake Fetcher Error: " + repr(e))
}

# export the do_unpack too. 
EXPORT_FUNCTIONS do_fetch do_unpack do_configure do_compile do_install
```

# defining `do_configure()`
```bash 

# add after do_patch {do_patch defined in patch.bbclass}
addtask configure after do_patch

# directory B. 
do_configure[dirs] = "${B}"

# defining the function
base_do_configure() {
	# after fetching and patching the source in directory, 
	# check stamp (HASH) or both. 
	# so if fetched and patched source HASH is same as current saved
	# HASH, we dont need to compile. 
	if [ -n "${CONFIGURESTAMPFILE}" -a -e "${CONFIGURESTAMPFILE}" ]; then
		if [ "`cat ${CONFIGURESTAMPFILE}`" != "${BB_TASKHASH}" ]; then
			cd ${B}
			
			# if hash different, run oe_runmake clean
			if [ "${CLEANBROKEN}" != "1" -a \( -e Makefile -o -e makefile -o -e GNUmakefile \) ]; then
				# clean the compiled file to compile again. 
				oe_runmake clean
			fi
			# -ignore_readdir_race does not work correctly with -delete;
			# use xargs to avoid spurious build failures
			find ${B} -ignore_readdir_race -name \*.la -type f -print0 | xargs -0 rm -f
		fi
	fi
	if [ -n "${CONFIGURESTAMPFILE}" ]; then
		mkdir -p `dirname ${CONFIGURESTAMPFILE}`
		echo ${BB_TASKHASH} > ${CONFIGURESTAMPFILE}
	fi
}
```

# `do_compile`

```bash

addtask compile after do_configure

# initalize working directory for do_compile, 
# build directory
do_compile[dirs] = "${B}"

base_do_compile() {
	# make sure, for compiling we have makefile or Makefile or GNUmakefile
	# for some, compile with ninja etc, they override the do_compile task
	if [ -e Makefile -o -e makefile -o -e GNUmakefile ]; then
		# call function which will compile.
		oe_runmake || die "make failed"
	else
		bbnote "nothing to compile"
	fi
}
```

# `do_install()`

```bash
 
addtask install after do_compile

# working directory as BUILD directory
do_install[dirs] = "${B}"
# Remove and re-create ${D} so that it is guaranteed to be empty
do_install[cleandirs] = "${D}"

# **shell no-op** — does absolutely nothing and returns success.
base_do_install() {
	:
}

########################################################################

# Every real recipe **must override** `do_install` to copy its built files into `${D}`. A typical recipe does:

do_install() {
    install -d ${D}${bindir}
    install -m 0755 mybinary ${D}${bindir}/mybinary
}

######################################################################


# Or for Makefile-based packages:
do_install() {
    oe_runmake install DESTDIR=${D}
}


```

# `do_build()`

`do_populate_sysroot` is the last real-work task — it takes installed files from `${D}` and copies them into the shared sysroot so other recipes can find headers/libraries. Once that's done, `do_build` fires.

```bash 
# add task of do_build after do_populate_sysroot
addtask build after do_populate_sysroot

# no execution task
do_build[noexec] = "1"

# recursive dependency
do_build[recrdeptask] += "do_deploy"

# empty task. we just need defination
do_build () {
	:
}
```

this tasm is no such actual task, just a graph node. last node of graph. we need it, as we always build towards `do_build` task and every graph starts to execute down. 
usually, we do `bitbake recipe-name`. it needs concrete task to execute, and that is do_build. 

`BB_DEFAULT_TASK = "do_build"` — BitBake always builds toward this task. Without it as a named node, there's nothing to point at.

`recrdeptask` = **recursive dependency task**.
Normal task dependencies are flat — "run A before B for this recipe." `recrdeptask` is graph-wide

> "Before `do_build` of **this** recipe is complete, `do_deploy` must have run on **this recipe AND every recipe it depends on, transitively.**"


From this bbclass, dependency graph of task which are defined are such as 

`do_fetch` -> `do_unpack` -> `do_configure` -> `do_compile` -> `do_install` -> `do_build`

Original graph of task is 
```
do_fetch
  → do_unpack
    → do_patch {not defined in base.bbclass}
      → do_configure
        → do_compile
          → do_install
            → do_package (from package.bbclass)
              → do_populate_sysroot (from staging.bbclass)
                → do_build   ← anchor point
```

`do_build` is default task for all recipes unless stated otherwise, and it depends on all normal tasks required to build the recipe. 
`bitbake core-image-minimal` with `-c` runs `do_build`, as value of `BB_DEFAULT_TASK ?= build`. if we change this variable, default task is changed. 
It exists to make sure everything else already happened. 
