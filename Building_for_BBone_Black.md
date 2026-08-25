# Building poky for generic machine. 

initialize the build setup. 
`./bitbake/bin/bitbake-setup init`
will init the setup. choose from option given through menu. initially we select to build minimial poky. 

From logs we can see what layers are fetched 
```
Fetching layer/tool repositories into /home/dipesh/projects/yocto/bitbake/bitbake-builds/initial_project_poky/layers
    bitbake
    openembedded-core           
    meta-yocto                  
    yocto-docs  
```

## Fragments 
	Fragments are reusable chunks of configurations. 
	These live in actual file in `layer's` `conf/fragments` directory

`bitbake-config-build list-fragments` : shows you what's currently switched on in your `toolcfg.conf`.


## Stock images 
An Image, in Yocto terms, is the actual build target — a top-level recipe (a `.bb` file) that pulls together package groups and produces a bootable rootfs plus whatever else the target machine needs.

`core-image-sato` specifically builds an image with Sato support — a mobile-style environment and visual theme, with X11 and applications like a terminal, editor, file manager, and media player.

---

## poky tiny 

`configs` : contain the configuration that are used. the current configurations. 
`layer` : the master folder which contains all the layers. 
				currently contains 
					bitbake itself. 
					meta-yocto : container repo for meta-poky and meta-yocto-bsp
					openembedded-core. 

```
- build 
- config 
	  - contains the configuration we choose while doing init. 
- layers 
	  - meta-yocto : umbrella directory for 2 layers. 
		    - meta-poky      : poky layer 
		    - meta-yocto-bsp : bsp layer 
		- openembedded-core : layer for building linux. 

```

