Open-embedded project started with group, which wanted to automate the compilation or building of custom embedded linux kernels.  they built the architecture and framework of recipes, modular approach etc. they built an engine, called bitbake, which reads the configuration files (.bb file, .conf file of this newly generated framework) and compile binaries, link etc. they are majorly inclusive. 
It supported range of hardware, range of software components.

Open Embedded Project started as open-source, community driven project. 
Its an generic Framework, which creates architecture agnostic custom Linux images for custom hardware. 
There is no limit on what type of hardware it supported. You define the hardware, use the framework and you can create YOUR custom linux image. 

# What the Framework contains ?
Framework has 3 major parts. 
1. bitbake : the task engine 
2. OpenEmbedded-core 
	1. core recipes. 
3. meta-openembedded 
	1. community layers. 
# Yocto and its relation with open embedded framework
Yocto is standalone project, which uses open-embedded framework to create custom linux images for certain hardware. 
Its an Industry Adoption. It borrows heavily from OpenEmbedded framework, or we can say, it sits on top of open embedded framework. 

# Component of Framework 
## bitbake 
Its just a task executor. can read more about it in [[bitbake]] page.

## OpenEmbedded-core 
Its the `oe-core` layer. The Layer which provides the basic Functionlity, which would generate bare-minimum linux distro image. 

In detail in [[oe-core]]

> minimal, architecture-agnostic base every other layer builds on

## meta-openembedded 

a separate git repository that is a collection of layers to supplement OE-Core with additional packages, each with its own designated maintainer

**The sub-layers you'll actually touch:**

- `meta-oe` — the general catch-all, biggest one, no clear specialty
- `meta-python` — Python module recipes (anything beyond the bare interpreter oe-core ships)
- `meta-networking` — intended to be a central point for networking-related packages and configuration, useful directly on top of oe-core, aimed at things like small routers or adding network services (ftp/tftp servers, VPN, etc.) to a device
- `meta-multimedia` — gstreamer plugins, ffmpeg, codecs
- `meta-filesystems` — btrfs-tools, squashfs-tools, ntfs-3g, exfat
- `meta-perl`, `meta-webserver` (apache2/nginx/lighttpd), `meta-gnome`/`meta-xfce` (desktop environments), `meta-initramfs`, `meta-selinux`

### **HOW it actually wires in — two mechanisms, and this is the part worth internalizing:**

1. **`LAYERDEPENDS`** — each sub-layer declares what it needs beneath it. A real example from `meta-filesystems`'s `layer.conf`: it declares dependence on `core openembedded-layer networking-layer` — i.e., it needs OE-Core, `meta-oe`, and `meta-networking` present before its own recipes will parse.
2. **`LAYERSERIES_COMPAT`** — each sub-layer also declares which Yocto release codenames it's compatible with (e.g. current development pins to two recent codenames). This is the enforcement point: you must check out the branch of `meta-openembedded` matching your `poky`/OE-Core branch. You're on scarthgap — pull `meta-openembedded`'s scarthgap branch, not master, or BitBake will refuse to parse it.