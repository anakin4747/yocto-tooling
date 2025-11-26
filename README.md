# yocto-tooling

# TODO

- add recommended tools
  - taskexp_ncurses.py
  - VSCode
  - Toaster

## Tools I will cover

- bitbake-layers
- bitbake-getvar
- bitbake
- recipetool
- runqemu
- oe-depends-dot
- oe-pkgdata-util
- oe-run-native
- buildhistory-collect-srcrevs

# Setup

The environment for this project will already be setup using `bitbake-setup`.

First let's source our environment:

```sh
cd bitbake-builds/
source distro_poky-master/build-tools/init-build-env
```

After sourcing the build script we have two folders added to our `PATH`
environment variable:

```sh
echo $PATH | tr ':' '\n'
```
```output
/home/ilab01/bitbake-builds/distro_poky-master/layers/openembedded-core/scripts
/home/ilab01/bitbake-builds/distro_poky-master/layers/bitbake/bin
/usr/local/bin
/usr/bin
/bin
/usr/local/games
/usr/games
```

Inside these two new folders are the tools and scripts we will be using
throughout this class.

## Getting Help

Most of the tools shown in this class accept a `-h` or `--help` flag to show
the usage. This is a great first step to get more help for how to use the
tools:

```sh
bitbake-layers -h
```

Note that subcommands can also accept a `-h` flag as well:

```sh
bitbake-layers show-recipes -h
```

If the help output doesn't answer your questions I recommend grepping your
layers for documentation on the tool in question:

```sh
grep -rn --color bitbake-layers ../layers/
```

To get even more information on a specific command I also recommend reading its
source code. They are typically straight-forward python scripts that are quite
readable. This makes them very approachable to read for a better understanding
of how the tool works.

For example to find the source code of `bitbake-layers`, you can use `which` to
get the path to the `bitbake-layers` script and then open it with your
favourite editor:

```sh
vim $(which bitbake-layers)
```

## bitbake-layers

The first command worth learning to manage a Yocto project is `bitbake-layers`.

```sh
bitbake-layers -h
```

The `-h` help output shows the subcommands which can be used with this script.

Firstly, we can use the `show-layers` subcommand to see what layers are
currently present in our project:

```sh
bitbake-layers show-layers
```

With this subcommand we can see what layers we have but this can also be used,
for example, to validate that a layer we added was correctly added and can be
seen by bitbake.

We want to have some layers and recipes to play with so first we will download
some layers from the `oe-layerindex`. Let's say we want `meta-python` to get some
python packages not available in the `meta` layer.

We can first use the `layerindex-show-depends` to see what dependencies
meta-python requires:

```sh
bitbake-layers layerindex-show-depends -b master meta-python
```

Note that we need to specify the master branch with `-b master` since this
class is using the master branch. Most likely this will not be needed for
earlier versions of Yocto and you can let `bitbake-layers` determine the best
branch for your configuration.

Now let's add the `meta-python` layer with the `layerindex-fetch` subcommand
(the --shallow option is to do a shallow git clone since we currently do not
need the git meta data of all the repos):

```sh
bitbake-layers layerindex-fetch -b master --shallow meta-python
```

Now we can see that the `meta-python` layer and its dependencies have now been
added to the project for us:

```sh
bitbake-layers show-layers
```

This is great for a couple reasons. We didn't have to go find the right commit
of `meta-python`. We didn't have to manually install it in our project. We
didn't need to manually add it to our `bblayers.conf`. We didn't have to repeat
that process for all of `meta-python`'s dependencies.

Layers need to be added to `/path/to/build/conf/bblayers.conf` for them to be
detected by bitbake. If a layer is downloaded but not listed in the
`bblayers.conf` or in the output of `show-layers` you can add these layers with
the `add-layer` subcommand:

```sh
bitbake-layers add-layer ../layers/meta-yocto/meta-yocto-bsp
bitbake-layers show-layers
```

We can also create our own layer with the `create-layer` subcommand:

```sh
bitbake-layers create-layer -e tuna -a -p 10 ../layers/meta-vader
bitbake-layers show-layers
```

This created a layer for me and filled out the boilerplate logic needed to
define a layer:

```sh
find ../layers/meta-vader/
../layers/meta-vader/
../layers/meta-vader/recipes-tuna
../layers/meta-vader/recipes-tuna/tuna
../layers/meta-vader/recipes-tuna/tuna/tuna_0.1.bb
../layers/meta-vader/conf
../layers/meta-vader/conf/layer.conf
../layers/meta-vader/COPYING.MIT
../layers/meta-vader/README
```

Notice that I set the priority with the `-p 10` flag and added the layer to
`bblayers.conf` with the `-a` flag to avoid the extra step of running
`add-layer` for the newly created layer. I also specified the name of the
example recipe to `tuna` as a suprise for later.

The next command can be used to display the recipes bitbake can see:

```sh
bitbake-layers show-recipes
```

This subcommand can be used, for example, to validate that a recipe we added
was correctly added and can be seen by bitbake.

The output of this command lists the recipe name and on the next line shows
what version is provided by what layer:

```output
...
zstd:
  meta                 1.5.7
```

The above output snippet shows that the `meta` layer provides version 1.5.7 of
`zstd`.

The `show-recipes` subcommand without any arguments can also show some recipes
that were detected but skipped due to incompatibilities:

```output
...
xf86-video-vmware:
  meta                 13.4.0 (skipped: incompatible with host aarch64-poky-linux (not
...
```

We can also specify a single recipe if we only want to query a single recipe:

```sh
bitbake-layers show-recipes xcb-util-errors
```

Or we can specify a wildcard if we want to group by a pattern:

```sh
bitbake-layers show-recipes "xcb-*"
```

The default output of this subcommand can be changed with a variety of flags.
For example, `-f` will show only full filenames instead and `-r` will only show
recipe names instead. These both do not display the layer and version. While
`-b` can be used to not display `(skipped)` markers.

Flags like `-l`, `-m`, and `-i` can be used to make more custom recipe queries.

The filter the recipes per layer you can add the `-l` flag to specify the
layer:

```sh
bitbake-layers show-recipes -l meta-python
```

The `-m` flag shows recipes only who have multiple definitions:

```sh
bitbake-layers show-recipes -m
```
<!-- TODO: determine why some single recipes end up in the output -->

The `-i` flag can be used to query only recipes that include a specific bbclass
or combinations of bbclasses:

```sh
# shows recipes which inherit kernel.bbclass
bitbake-layers show-recipes -i kernel

# shows recipes which inherit systemd.bbclass and meson.bbclass
bitbake-layers show-recipes -i systemd,meson
```

In the same way we have been able to validate our layers and recipes were correctly added
with `show-layers` and `show-recipes`, we can do the same for bbappends with
the `show-appends` subcommand:

```sh
bitbake-layers show-appends
bitbake-layers show-appends linux-yocto
```

The same goes for validating the addition of a machine configuration with the
`show-machines` subcommand which can also accept a `-l` flag to specify the
layer:

```sh
bitbake-layers show-machines
bitbake-layers show-machines -l meta-yocto-bsp
```

The `show-overlayed` subcommand can be used to show recipes defined in multiple
layers:

```sh
bitbake-layers show-overlayed
```

The `show-cross-depends` subcommand shows dependencies between recipes that
cross layer boundaries:

```sh
bitbake-layers show-cross-depends
```

This is great for determining what layers your layer depends on to correctly
set `LAYERDEPENDS_meta-<layer>` in layer.conf.

Turns out I decided I do not want `meta-python` and `meta-yocto-bsp` so I would
like to remove them and their dependencies. This can be done with the
`remove-layer` subcommand. Note that I will also remove `meta-oe` as it was
only included due to `meta-python`'s dependency on it:

```sh
bitbake-layers remove-layer meta-python meta-oe meta-yocto-bsp
bitbake-layers show-layers
```

## bitbake-getvar

There are tools for inspecting deeper inside the build environments of a
recipe. One excellent tool to dig deeper is `bitbake-getvar`. We can use it to
get the value of bitbake variables in the build configuration or for specific
recipes.

First we will use it to confirm our `DISTRO` and `MACHINE` are correctly set:

```sh
bitbake-getvar MACHINE
NOTE: Starting bitbake server...
#
# $MACHINE [3 operations]
#   set /home/ilab01/bitbake-builds/distro_poky-master/build-tools/conf/local.conf:29
#     [_defaultval] "qemux86-64"
#   set /home/ilab01/bitbake-builds/distro_poky-master/build-tools/conf/local.conf:251
#     "qemuarm64"
#   set /home/ilab01/bitbake-builds/distro_poky-master/layers/openembedded-core/meta/conf/documentation.conf:274
#     [doc] "Specifies the target device for which the image is built. You define MACHINE in the conf/local.conf file in the Build Directory."
# pre-expansion value:
#   "qemuarm64"
MACHINE="qemuarm64"
```

```sh
bitbake-getvar DISTRO
NOTE: Starting bitbake server...
#
# $DISTRO [2 operations]
#   set /home/ilab01/bitbake-builds/distro_poky-master/layers/openembedded-core/meta/conf/bitbake.conf:788
#     [_defaultval] "nodistro"
#   set /home/ilab01/bitbake-builds/distro_poky-master/layers/openembedded-core/meta/conf/documentation.conf:140
#     [doc] "The short name of the distribution. If the variable is blank, meta/conf/distro/defaultsetup.conf will be used."
# pre-expansion value:
#   "nodistro"
DISTRO="nodistro"
```

The output of these two commands show us the value of the specified variable as
well as the history of how this variable has been set. For example, the default
value of MACHINE was "qemux86-64" at line 29 in `conf/local.conf` but got
changed to "qemuarm64" at line 251 in `conf/local.conf`. As for DISTRO, it is
still the default value of "nodistro" which was set by like 788 in
`bitbake.conf`.

We can also specify a recipe with `-r` to get recipe specific variables:

```sh
# maybe you want to quickly see where you are getting your kernel sources from
bitbake-getvar -r virtual/kernel SRC_URI
# but lets see that unexpanded value
bitbake-getvar -r virtual/kernel -u --value SRC_URI
# see varflags on variables
bitbake-getvar -f doc --value SRC_URI
```

This is a great tool for debugging the assignment of specific variables.

## bitbake

One of the most used commands when interacting with Yocto is the `bitbake`
command.

The follow command builds the recipe called `core-image-minimal`:

```sh
bitbake core-image-minimal
```

Runnning `bitbake` without any flags will stop once it runs into an error.
Often when running an unattended build this behaviour is underised. To have
`bitbake` continue to build as much as it can even when there are errors you
can use the `-k` or `--continue` flag:

```sh
bitbake -k core-image-minimal
```

If you just want to validate that all recipes are syntatically correct you can
run bitbake with the `-p` flag to only parse the recipes:

```sh
bitbake -p core-image-minimal
```

Or for a more thorough validation of the setup you can run bitbake with the
`-n` flag to perform a dry run of the build:

```sh
bitbake -n core-image-minimal
```

Often times we will want to explicitly run a specific task within a recipe.
This can be done with the `-c` flag. The best task to run to learn more about
this functionality is the `listtasks` task that every recipe has by default:

```sh
bitbake -c listtasks busybox
```

```output
...
do_build                              Default task for a recipe - depends on all other normal tasks required to 'build' a recipe
do_checkuri                           Validates the SRC_URI value
do_clean                              Removes all output files for a target
do_cleanall                           Removes all output files, shared state cache, and downloaded source files for a target
do_cleansstate                        Removes all output files and shared state cache for a target
do_collect_spdx_deps
do_compile                            Compiles the source in the compilation directory
do_configure                          Configures the source by enabling and disabling any build-time and configuration options for the software being built
...
do_devshell                           Starts a shell with the environment set up for development/debugging
do_diffconfig                         Compares the old and new config files after running do_menuconfig for the kernel
do_fetch                              Fetches the source code
do_install                            Copies files from the compilation directory to a holding area
do_listtasks                          Lists all defined tasks for a target
do_menuconfig                         Runs 'make menuconfig' in the compilation directory
...
```

All of these tasks listed can be passed as an argument to `-c` with or without
the `do_` prefix.


To only run the compile task of `busybox`:

```sh
bitbake -c compile busybox
```

Sometimes bitbake may determine that a task did not need to run. To force a
task to run use the `-f` flag:

```sh
bitbake -f -c compile busybox
# or equivalently
bitbake -fc compile busybox
```

Some tasks are useful for debugging. Such as the `devshell` task. This task
will place you in a shell in the source code of the recipe with the same
environment that is used to build the recipe. The `pydevshell` task will do the
same but instead puts you in a python repl:

```sh
bitbake -c devshell busybox
bitbake -c pydevshell busybox
```

Recipes can be cleaned to varying degrees with the `clean*` tasks:

```sh
bitbake -c clean busybox
bitbake -c cleansstate busybox
bitbake -c cleanall busybox
```

Note that you should avoid `cleanall` if you do not wish to refetch the source
code.

Another extremely useful flag to bitbake is the `-e` flag. This is an
incredible tool for debugging as it prints the entire environment of a recipe
or build. This is essentially the same thing as `bitbake-getvar` but it applies
to every variable and task in the recipe's environment. In the same manor as
`bitbake-getvar` it also shows the history of how a task or variable was
defined. It produces a very large file so its best to redirect it to a file or
pipe it to `tee`. I recommend using a file extension that will tell your editor
that it is best highlighted as a shell script such as `.sh` or `.env`:

```sh
bitbake -e core-image-minimal | tee image.env
```

```sh
bitbake --runall fetch world
bitbake --runall fetch core-image-minimal
```

```sh
bitbake -g core-image-minimal
```

### taskexp_ncurses

```sh
bitbake -g -u taskexp_ncurses zlib acl
```

## recipetool

```sh
recipetool edit example
```

```sh
# kind of annoying that the path must be specified even though it doesn't need
# to be for the edit subcommand
recipetool setvar \
    ../src/meta-vader/recipes-example/example/example_0.1.bb \
    SUMMARY "A super cool example recipe"

# whats cool about this is you can create patches to apply to the layer to
# setvar, unfortunately this situation isn't common enough for a command like
# this to become second hand knowledge. Pretty small use-case.
recipetool setvar \
    --patch \
    ../src/meta-vader/recipes-example/example/example_0.1.bb \
    SUMMARY "A super cool example recipe"
```

```sh
mkdir /tmp/example-source-code
cat << EOF > /tmp/example-source-code/complex.sh
#!/bin/sh
echo "super complex shell script"
EOF
recipetool create /tmp/example-source-code/complex.sh \
    -o ../src/meta-vader/recipes-example/complex-script.bb
```

```sh
recipetool newappend -w ../src/meta-vader virtual/kernel
```

```sh
# requires build first
cat << EOF > hosts
127.0.0.1        localhost
EOF
recipetool appendfile ../src/meta-vader /etc/hosts hosts
```

```sh
zcat /proc/config.gz > this_defconfig
recipetool appendsrcfile ../src/meta-vader virtual/kernel this_defconfig \
    arch/x86/configs/this_defconfig
# although your kernel provider may have their own way to implement this
```

## runqemu
<!-- ~/src/yocto-tooling/videos/runqemu.mkv -->

```sh
runqemu slirp qemux86-64 nographic
```

```sh
runqemu slirp qemux86-64 nographic \
    qemuparams="-s -S" \
    bootparams="nokaslr"
```

```vim
Termdebug /home/kin/yocto-project/build/tmp/work/qemux86_64-poky-linux/linux-yocto/6.6.96+git/linux-qemux86_64-standard-build/vmlinux
```

```.gdbinit
target remote :1234
set substitute-path /usr/src/kernel /home/kin/yocto-project/build/tmp/work-shared/qemux86-64/kernel-source/
b start_kernel
```

## oe-depends-dot
<!-- ~/src/yocto-tooling/videos/oe-depends-dot.mkv -->

```sh
bitbake -g core-image-minimal

oe-depends-dot -k busybox -w ./task-depends.dot

oe-depends-dot -k busybox -d ./task-depends.dot
```

## oe-pkgdata-util
<!-- ~/src/yocto-tooling/videos/oe-pkgdata-util.mkv -->

```sh
oe-pkgdata-util find-path /etc/security/namespace.conf
```

```sh
oe-pkgdata-util lookup-recipe libpam-runtime
```

```sh
oe-pkgdata-util list-pkgs libpam\*
```

```sh
oe-pkgdata-util list-pkg-files libpam
```

```sh
oe-pkgdata-util package-info libpam
```

## oe-run-native
<!-- ~/src/yocto-tooling/videos/oe-run-native.mkv -->

```sh
bitbake -c addto_recipe_sysroot ninja-native
oe-run-native ninja-native ninja -h
```

## buildhistory-collect-srcrevs
<!-- ~/src/yocto-tooling/videos/buildhistory-collect-srcrevs.mkv -->

```sh
buildhistory-collect-srcrevs -a
buildhistory-collect-srcrevs >> conf/local.conf
```

## Toaster

```sh
```

## VSCode


