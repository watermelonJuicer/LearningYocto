provides 
```
addtask populate_sysroot after do_install
```

`do_populate_sysroot` gives us file for compilation of given recipe. 

Suppose out application `mytool` is dependent upon two library files for compilation (which we compile through another recipe). 
At time of compilation of `mytool`, we would need .so and header files from these dependencies to be at one place, so we can instruct compiler to look into given directory.  
This step is achieved by `do_populate_sysroot` and recipe's own `do_configure`

my_tool dependent on 
	- `libgreet`
	- `libfetch`
Before compilation of `mytool`, `libgreet` and `libfetch` both will be compiled, and their `do_install` step will be run. 

do_install -> creates -> `{D}` 
`{D}` is used by `do_populate_sysroot` and `do_package`

`do_populate_sysroot`
- for each recipe, go through its `{D}` and figure out files, which will be required by compiler (.so, headers etc) and discard file like documentation etc. 
- copy the files into some directory (unique for each recipe)

for `libgreet` and `libfetch`,  go through their `{D}` and paste files into unique folders
```
sysroots-components/cortexa8hf-neon-mx/libgreet/
	├── usr/
		├── lib/
			├── libgreet.so.1.0.0
			├── libgreet.so.1
			├── libgreet.so
			└── pkgconfig/
			└── libgreet.pc
		└── include/
			└── greet.h
	└── sysroot-providers/
		└── libgreet ← manifest: exactly what this recipe staged
		
		

sysroots-components/cortexa8hf-neon-mx/libfetch/
	├── usr/
		├── lib/
			├── libfetch.so.2.0.0
			├── libfetch.so.2
			├── libfetch.so
			└── pkgconfig/
			└── libfetch.pc
	└── include/
		└── fetch.h
	└── sysroot-providers/
		└── libfetch
```

## `mytool` compilation

During mytool compilation, in its recipe file the dependecies of `libgreet` and `libfetch` is listed. 
`do_configure` will figure out dependencies of `mytool` compilation, and will go through files of depenedencies in `sysroot-component` directory, and paste into common sysroot folder for compilation shared by all dependency. 

> For each recipe, the dependency folder is unique, but that folder is shared by all dependency file. 

In sub-folder of mytool
`${WORKDIR}/recipes-sysroot/`

```

# for dependency for cross compilation
${WORKDIR}/recipes-sysroot/
	└── usr/
		├── lib/
		│ ├── libgreet.so.1.0.0
		│ ├── libgreet.so.1
		│ ├── libgreet.so
		│ ├── libfetch.so.2.0.0
		│ ├── libfetch.so.2
		│ ├── libfetch.so
		│ └── pkgconfig/
		│ ├── libgreet.pc
		│ └── libfetch.pc
	└── include/
	├── greet.h
	└── fetch.h


# if some dependency is dependent upon native compilation
${WORKDIR}/recipes-sysroot-native/
	└── usr/
	└── bin/
	└── codegen
```


For mytool too, if output generates files required for compilation, same will be applied for mytool too. 