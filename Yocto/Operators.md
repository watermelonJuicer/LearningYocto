# Operators 

> Anything defined in conf files is global. Anything defined in recipes, is local. 

## hard assignment
```bash 
# Basic variable setting 
# below is hard assignment. if space in " raspberrypi3", space will be considerd in value name too. 
# hard assignment : Assignment occurs as soon as below hard assignment line is parsed. 
MACHINE = "raspberrypi3"

# find value of variable.

# below command shows global variable values defined in local files. (local.conf, bblayers.conf etc)
bitbake -e 

# for checking values of variables in receipe, use command. 
bitbake <receipe-name> -e 

# for looking value of variable which is global, below command will fetch and parse global values from global configuration etc. 
bitbake -e | grep "<VARIABLE="

# will give all occurance of variable
bitbake -e | grep "VARIABLE"

# will give value of variable, where it is assigned. 
bitbake -e | grep "VARIABLE="

# for looking at variable values in receipes, 
bitbake < receipe-name > -e | grep ^VARIABLE_NAME=
```
## soft variable Assignment
```bash 
# ?= used for soft assignment.

# allows us to define variable if it is undefined  when statement is parsed. 
# if value is already defined, then ?= assignment doesn't work. 

MACHINE ?= "qemuarm"

# If machine is already set before this statement, then above value is not assigned. 
# if machine is not set, above value is assigned. 


# in presence of hard assignment, the hard assignment value is taken, otherwise SoftAssignment.


# incase of multiple soft assignment, first one will be assigned. 
MACHINE ?= "qemux86"
MACHINE ?= "qemuarm"

# will give us MACHINE=qeumx86 
# if there is no hard assignment before or after this. 

# after this, if we do 
bitbake -e | grep MACHINE= 
```
## Weaker default value
```bash 
# ??= 
# Assignment is made at end of parsing process.
# when multiple ??= exists, last one is used as default. 

MACHINE ??= "qemux86"
MACHINE ??= "qemuarm"

# after this, if there is no MACHINE hard assignment set, 
# MACHINE = "qemuarm"
```
## precendence order or priority order

> `=` overides `?=` which over rides `??=`

> Bitbake joins any line ending in backslash character `\`. 
```bash 

# BBLAYERS in bblayer.conf 
# ending with \, will join the next line values. 
BBLAYERS ?= " \
	.../meta \
	.../poky/meta-poky \
	.../poky/meta-yocto-bsp \
	"
```

## Variable Expansions
```bash 

A = "hello"
B = "${A} World"

bitbake -e | grep ^B=
# $ hello world

##---------------------------------------------------------------------------##

# expansion is delayed, until the variable is actually used. 
A = "${B} hello"
B = "${C} world"
C = "Linux"

bitbake -e | grep ^A=
# $ Linux world hello

##---------------------------------------------------------------------------##
# if string is not found, its kept as it is. 

A = "${B} hello"
B = "${C} world"

bitbake -e | grep ^A=
# A="\${C} world hello"

```
## Immeditate variable Expansion 

':=' operator results in variable contents being expanded immediately rather then expanded when its being actually used. 

```bash 

A = "11"
B = "B:${A}"
A = "22"

# B = B:22
# as latest value of A would be 22, at the end, thats the value 
# getting stored in B. 

##------------------------------------------------------------------# 

A = "11"
B = "B:${A}"
A = "22"
C := "C:${A}"
A = "44"

### output in the end. 
B = B:44  # as in last, A = 44
C = C:22  # immeditate variable assignment, so while C was being assgined, A value was 22. hence C:22. 

```
## Appending and prepending Operators
> `+=` and `.=` are appending operators. 
> `=+` and `=.` are prepending operators. 

`+=` adds space automatically. 

```bash 

A = "hello"
A += "world"

$ A = hello world
##########################################

A = "hello"
A .= "world"

$ A = helloworld

##########################################

A = "hello"
A =+ "world"

A = "world hello"

##########################################

A = "hello"
A =. "world"
A = "worldhello"
```
## overriding style syntax
using `_append` or `_prepend` after variable name, also appends and prepends value. 
```bash 

A = "hello"
A_append = " world"

$ A = "hello world"
# it doesnt add spaces before or after. 


A = "world"
A_prepend = "hello"
$ A = "helloworld"

```
## Removal syntax 
adding `_remove` will remove values, from all occurances of that value from that variable. 
```bash 

FOO = "123 456 789 123456"
FOO_remove = "123"

bitbake -e | grep ^FOO=
$ "456 789 123456"
```

Advantage of using overriding style syntax, `_append`, `_prepend`, `_remove` is that they gurantees operation. 


