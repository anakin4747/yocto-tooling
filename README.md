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
- devtool
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
source distro_poky-master/build/init-build-env
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
the usage. This is a great first step to get more help for how to use the tools
mentioned in this class:

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
source code. They are typically straight-forward single-file python scripts
that are quite readable. This makes them very approachable to read for a better
understanding of how the tool works.

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

The `show-layers` subcommand to see what layers are currently present in our
project:

```sh
bitbake-layers show-layers
```

With this subcommand we can see what layers we have but this can also be used,
for example, to validate that a layer we added was correctly added and can be
seen by bitbake.

Now let's add the `meta-python` layer with the `layerindex-fetch` subcommand
(the --shallow option is to do a shallow git clone since we currently do not
need the git meta data of all the repos):

```sh
bitbake-layers layerindex-fetch -b master --shallow meta-python
```

<!-- TODO: add creating custom layer -->

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
bitbake-layers show-recipes -l meta-poky
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

# shows recipes which inherit systemd.bbclass and cmake.bbclass
bitbake-layers show-recipes -i systemd,cmake
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
<!-- TODO: Could be setup so that this can actually be shown? maybe? -->

The `show-cross-depends` subcommand shows dependencies between recipes that
cross layer boundaries:

```sh
bitbake-layers show-cross-depends
```
<!-- TODO: Could be setup so that this can actually be shown? maybe? -->

This is great for determining what layers your layer depends on to correctly
set `LAYERDEPENDS_meta-<layer>` in layer.conf.

## bitbake-getvar

Now that my build is setup I can start building, but first let's just double
check that my template has taken affect by checking that my MACHINE and DISTRO
are properly set.

```sh
bitbake-getvar MACHINE
bitbake-getvar DISTRO
# maybe you want to quickly see where you are getting your kernel sources from
bitbake-getvar -r virtual/kernel SRC_URI
# but lets see that unexpanded value
bitbake-getvar -r virtual/kernel -u --value SRC_URI
# see varflags on variables
bitbake-getvar -f doc --value SRC_URI
# most useful of them all
bitbake-getvar -h
```

Great for investigating if setting a variable was redundant.

## bitbake
<!-- ~/src/yocto-tooling/videos/bitbake.mkv -->

Most used bitbake commands:

```sh
bitbake core-image-minimal
bitbake -k core-image-minimal
bitbake -n core-image-minimal
```

```sh
bitbake -c listtasks busybox
bitbake -c compile busybox
bitbake -c devshell busybox
bitbake -c pydevshell busybox
bitbake -c clean busybox
bitbake -c cleansstate busybox
bitbake -c cleanall busybox
```

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

## recipetool
<!-- ~/src/yocto-tooling/videos/recipetool.mkv -->

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

## devtool
<!-- ~/src/yocto-tooling/videos/devtool.mkv -->

```sh
cat /tmp/example-source-code/complex.sh
# creates this recipe in the workspace for you to edit
devtool add complex-script /tmp/example-source-code/complex.sh
devtool rename complex-script simple-script
devtool finish simple-script meta-vader
```

```sh
# when I want to patch the kernel
devtool modify virtual/kernel
devtool menuconfig linux-yocto

CONFIG_GDB_SCRIPTS=y
CONFIG_WATCHDOG=n

# highlight the difference between devshell and devtool
bitbake -c devshell virtual/kernel
make scripts_gdb
devtool finish linux-yocto ../src/meta-vader/
```

```sh
devtool status
```

```sh
devtool reset -a
```

```sh
devtool finish --force-patch-refresh virtual/kernel
```

```sh
devtool latest-version virtual/kernel
devtool check-upgrade-status virtual/kernel

devtool check-upgrade-status busybox
devtool upgrade busybox
```

```sh
devtool search ostree
# limited without building ahead of time
# searches locally not online lame
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
