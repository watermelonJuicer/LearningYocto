
# To clean and build from scratch

`bitbake -c clean busybox` : $WORKDIR + everystam file for the recipe. 
`bitbake -c cleanslate busybox` : Everything -c clean doesn plus sstate metadata also
`bitbake -c cleanall busybox` : Everything cleanstate does plus fetched source.


List all task available for busybox. 
```
bitbake -c listtasks busybox  
```

The `-c` argument accepts the task name with or without the `do_` prefix (`-c compile` and `-c do_compile` are equivalent). It doesn't blindly re-run the whole chain — bitbake still checks stamps/hashes and skips anything already up to date; it just moves the _target_ of the build from the recipe's default (`do_build`) to the task you named.

# Finding out directories via `bitbake -e`

```bash
bitbake -e busybox | grep -E '^(S|B|D|WORKDIR|T|PKGD|PKGDEST)='


# S : unpacked source, input to do_configre/do_compile
# B : output of build or do_compile 
# D : do_install output 
# T : log and generated script for every task 
# WORKDIR : parent of all above directory
# PKGD : do package intermediate split root 
# PKGDEST : final per-package split trees
```

# Finding the find resolved script for given step 

Before even start executing, bitbake resolves all steps. For example `do_package`, it will go through given recipe bb file, and package.bbclass file and resolve the appends, prepends, the variables etc and get final script. The final script will be just executed in execution step. 
We can checkout that final script only, using following way, 

inside each recipe's own `WORKDIR/temp/` directory. Under `WORKDIR/temp` we get two files per task-run — the generated script itself and a matching log, each named with the task and the process ID that produced them , 
So we will se `run.do_package` and `log.do_package`

Reading run.do_packge, its resolved script. latest one. 
```bash
<TMPDIR>/work/<arch>-poky-linux-gnueabi/busybox/<pv>-<pr>/temp/run.do_package
```

The file **is** the fully resolved script — every `:append`/`:prepend`/override already merged in textually, all variables already expanded.

force it to regenerate without needing the task to be "out of date":

```
bitbake -c package -f busybox
```

`-f` forces re-execution even if BitBake thinks nothing changed.
# Adding own debug to log files. 

Every line a shell task function writes to stdout/stderr is captured verbatim into that task's `log.do_<task>.<pid>` — that's the whole logging mechanism, there's no separate "debug channel" you need to opt into. So:

- In a **shell function** (`do_prepare_config`, `do_configure`, etc.): just `echo "MYDEBUG: S=${S} foo=$foo"` — it lands directly in `log.do_<task>.<pid>` at the point it executes.
- For **shell tracing** of every command bitbake already wires up `set -x`-equivalent tracking (see `bb_bash_debug_handler` at the top of `run.do_configure`) — that's why failures show file:line backtraces automatically.
- In a **python function** (the anonymous `python () {}` blocks): use `bb.note("...")`, `bb.warn("...")`, or `bb.plain("...")` — these go to the task log too (and `bb.warn`/`bb.note` also surface in the main bitbake console output).
- `bbdebug 1 "msg"` inside a shell function only shows up if you run bitbake itself with `-D` (each `-D` raises verbosity one level).

Bitbake always maintains a symlink without the PID suffix pointing at the most recent run:

```
log.do_configure -> log.do_configure.76117
run.do_configure -> run.do_configure.76117
```

Just read `log.do_configure` / `run.do_configure` (no suffix) and you always get the latest, full stop.

**`log.task_order`** — a single authoritative, timestamped ledger of every task ever executed in that `WORKDIR`, in order, with PID. From your actual file:

```
20260916-073031.790638 do_configure (67893): log.do_configure.67893
20260916-073204.066598 do_configure (69403): log.do_configure.69403
20260916-073951.060310 do_configure (71853): log.do_configure.71853
20260916-074104.593331 do_configure (73137): log.do_configure.73137
20260916-074213.496933 do_configure (74477): log.do_configure.74477
20260916-074632.774487 do_configure (76117): log.do_configure.76117   ← last one, matches the symlink
20260916-075351.522993 do_compile   (78224): log.do_compile.78224
```


# Exporting graph for recipe. 

Export whole graph of given recipe
```bash 
bitbake -g busybox


# convert to svg format 
dot -Tpng task-depends.dot -o task-depends.png


# interactive viewew for dot graphs. 
xdot task-depends.dot
```


Filter to just recipe we care about 
```bash 
bb-depends-dot task-depends.dot -tr busybox > busybox-subgraph.dot   # transitive deps of busybox
dot -Tsvg busybox-subgraph.dot -o busybox-subgraph.svg
```


# Understand function types. 

Function defined as `python <function_name> ()` are exectued when called, not during parsing time. 
Anonymous function `python ()` are executed during parsing time. 
`def` defined functions are exectued when called. 