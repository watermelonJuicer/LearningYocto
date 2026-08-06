Bitbake knows nothing about linux, how to build it from scratch etc. 

> Its just a task executor. Resolves DAG (Directed graph of some type) and executes task. 

BitBake is fundamentally a generic task execution engine that runs shell and Python tasks efficiently and in parallel while respecting complex inter-task dependencies — OpenEmbedded is just one user of that core, which happens to build embedded Linux stacks on top of it.

Bitbake knows 3 things.
1. how to parse 3 files. (`.conf / .bb / .bbclass`) into data-structure called datastore.
	1. `.conf` files are parsed first. `.conf` files job is to define global policy before any recipe exists. 
	2. .bb and .bbclass files are more per recipe and task oriented. 
2. How to build dependency graph out of tasks. 
3. How to execute this task graph in parallel. 

Everything we associate with `yocto` -- `do_compile`, sysroot, packages, images is `metadata` OE-core feeds into bitbake. 