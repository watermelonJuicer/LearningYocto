A bbclass file is same as .bb file, but its purely intended for inhertiting and code sharing. 
`.bb` file is intended for execution. we do `bitbake <bb-file>` and we execute the recipe. 

`.bbclass` file, is purely for code duplication and inheriting code. Existing purely for code-reuse. 



--
all the tasks, defined in bitbake, fetch -> unpack -> patch -> compile -> package
can be mapped one on one with python functions, do_fetch(), do_unpack() etc. 
these functions are shared, and are all defined in some or other bbclasses file. these bbclasses file and 
inherited in bbfiles, and then these functions are used. 

BitBake itself is a Python program. It parses every .bb/.bbclass/.conf file into an in-memory datastore object (call it d — you'll see this exact variable name everywhere in real class code) that holds every variable and every task definition.


A shell task (do_x() { ... }) is not interpreted by Python at runtime. BitBake takes the shell body, expands every ${VAR} using the datastore, and writes out an actual standalone shell script to disk — then hands it to a real /bin/sh subprocess. Python's job ends at code generation.

You can literally see this: after any build, the generated, fully-expanded script for a task sits at

`tmp/work/<arch>/<recipe>/<version>/temp/run.do_compile`


why we dont do inherit base class, so we can use functions like do_compile(), do_package() etc. 

The base.bbclass file is always included [for every recipe]. Other classes that are specified in the configuration using the INHERIT variable are also included.

base.bbclass is always included in all the recipes. 
