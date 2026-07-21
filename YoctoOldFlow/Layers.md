Why we need to create layer ?

we dont edit upstream layers, as it can complicate future updates. 

Depending on type of layer
1. If layer adds support for machine, add machine configuration to `conf/machine/`.
2. If layer adds distro policy, add distro configuration to `/conf/distro`
3. If layer introduces new receipe, put receipes in receipe-* sub-directories. 

## Manually creating custom layer 
```bash 

# create a layer folder in source of poky. or anywhere. 
# at last you will give directory or location of file. 

mkdir meta-jdcrlayer

# create conf directory inside customlayer. 
mkdir meta-jdcrlayer/conf

# copy conf/layer.conf file from reference already present layer to 
# newly created layer. 
cp ./meta-yocto-bsp/conf/layer.conf ./meta-jdcrlayer/conf/

# edit the conf pasted in meta-jdcrlayer/conf/layer.conf. 
# change name, version etc. 

# add layer directory in bblayer.conf. [<prj>/conf/bblayer.conf]

# check layer is added by command 
bitbake-layers show-layers
 
```

## creating new custom layer using `bitbake` command

```bash 

# for creating layer, and related functionality. 
bitbake-layers create-layer --help 

# create layer by name jdcr_mylayer2
# default priority of layer being created 6.

# bitbake-layers create-layer <locatio for layer to be made >
bitbake-layers create-layer ../layers/jdcr_mylayer2

# add that layers using command, bitbake-layers add-layer <location of layer directory>
bitbake-layers add-layer ../layers/jdcr_mylayer2

# list all layers. 

```

## Understanding `layer.conf`

### `BBPATH`
```bash 
BBPATH .= ":${LAYERDIR}"
```
Bitbake's search path for 
1. `.bbclass` files.
2. `.conf` includes
3. anything referenced via `require/inherit`. 
`.=` appends layer directory, to whatever `BBPATH` already exists.  (built up from all the layers processed so far).

### `LAYERDIR`
	`LAYERDIR` is not something we set explicitly. 
	BitBake auto-populates it to the absolute path of the layer, at the point `layer.conf` is being parsed. 
	We only _reference_ it.

### `BBFILES`
```bash 
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"
```
This is the **most important line** 
It's the actual glob pattern telling BitBake where our recipes (`.bb`) and appends (`.bbappend`) physically live.
If we get this wrong (wrong depth, wrong wildcard), BitBake will simply never see our recipes, and we'll get no error — just silent absence.

The conventional Yocto structure is:
`recipes-<category>/<recipe-name>/<recipe-name>.bb`

### `BBFILE_COLLLECTIONS`
registers your layer's collection name into the global list of active layers

### `BBFILE_PATTERN_my_layer`
A regex used to test which files (matched via BBFILES globally, across all layers) actually "belong" to this collection. Since BBFILES is a flat global list contributed to by every layer, BitBake needs a way to attribute each matched file back to its owning layer for priority/override purposes — this pattern is that attribution mechanism. Anchoring on ^${LAYERDIR}/ is standard and rarely needs changing.

### `BBFILE_PRIORITY_mylayer`
This is the conflict resolution priority, a integer (higher = higher priority). It matters in two scenarios:

Same recipe (same PN) provided by two layers — the higher-priority layer's version is preferred by default.
Multiple .bbappend files targeting the same recipe — they're applied in priority order (lowest → highest), so a higher-priority layer's append effectively has the "final say" in ordering.

### `LAYERDEPENDS_mylayer`

```bash 
LAYERDEPENDS_mylayer = "core openembedded-layer"`

# we can also pin versions. 
LAYERDEPENDS_mylayer = "core:5"

```
Declares hard dependencies on other layers' collection names (not directory names — collection names, as declared by their BBFILE_COLLECTIONS). If a listed dependency isn't present in bblayers.conf, BitBake errors out at parse time instead of failing mysteriously later when a require/inherit can't be resolved. core refers to OE-Core's collection (from meta/conf/layer.conf) and is almost always a dependency.

### `LAYERSERIES_COMPAT_mylayer`
```bash 
LAYERSERIES_COMPAT_mylayer = "scarthgap"
```
Lists the Yocto release codenames your layer has been validated against (e.g. kirkstone, langdale, mickledore, nanbield, scarthgap).

### FULL Example
```bash 
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "mylayer"
BBFILE_PATTERN_mylayer = "^${LAYERDIR}/"
BBFILE_PRIORITY_mylayer = "6"

LAYERDEPENDS_mylayer = "core"
LAYERSERIES_COMPAT_mylayer = "scarthgap"
```

