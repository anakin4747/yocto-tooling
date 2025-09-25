# yocto-tooling

## Tools I will cover

- [x] bitbake-layers
- [x] setup-layers
- [x] setup-build
- [x] bitbake-getvar
- [x] bitbake
- [x] recipetool
- [ ] devtool
- [ ] runqemu
- [ ] oe-depends-dot
- [ ] oe-pkgdata-util
- [ ] oe-run-native
- [ ] buildhistory-collect-srcrevs

To begin I want to show how to setup Yocto using standard yocto tooling (ie not
using kas or repo).

Yocto project layout I chose when using official Yocto tools:
```sh
yocto-project (not source controlled)
├── build     (not source controlled)
└── src
    ├── meta-openembedded (separate git repo) (third party layer)
    ├── poky              (separate git repo) (third party layer)
    └── meta-vader        (separate git repo) (our bootstrap layer)
```

What I personally prefer for a Yocto project layout:
```sh
yocto-project (git repo)
├── build     (not source controlled)
└── src
    ├── meta-openembedded (separate git repo) (third party layer)
    ├── poky              (separate git repo) (third party layer)
    └── meta-vader (source controlled as part of yocto-project repo)
```

I wasn't able to get my preferred layout with the official tools because it
would treat the top level git repo as a layer so I couldn't figure out how to
use `setup-layers` without it creating `yocto-project/yocto-project`.

## Setting up a Yocto project from scratch with official Yocto tools

```sh
rm -rf ~/yocto-project*
mkdir ~/yocto-project
cd ~/yocto-project
```

First things first we need `poky`:

```sh
git clone git://git.yoctoproject.org/poky -b scarthgap --depth 1 src/poky
```

Gunna cheat a lil with `cqfd` since I am on Arch:

```sh
cp -r ~/src/yocto-tooling/.cqfd* .
cqfd init
cqfd shell
```

```sh
source src/poky/oe-init-build-env
```

Let's see what layers we have right now:

```sh
bitbake-layers show-layers
```

We can add a layer from the oe-layer index:

```sh
bitbake-layers layerindex-show-depends meta-python
bitbake-layers layerindex-fetch --shallow --fetchdir ../src meta-python
bitbake-layers show-layers
bitbake-layers show-recipes -l meta-python
```

Just to show some more bitbake-layers commands I will delete some layers:

```sh
bitbake-layers remove-layer meta-python meta-oe
bitbake-layers show-layers
```

Now lets add a custom layer but we need to track it with git so that
setup-layers treats it as a layer:

```sh
bitbake-layers create-layer ../src/meta-vader
git -C ../src/meta-vader init
git -C ../src/meta-vader add .
git -C ../src/meta-vader commit -m "initial commit"
```

Now we need to add our custom layer to our bblayers.conf, we can do this
manually or with `bitbake-layers`.

```sh
bitbake-layers show-layers
bitbake-layers add-layer ../src/meta-vader
bitbake-layers show-layers
```

Let's set the machine and distro in local.conf:

```sh
cat << EOF > conf/local.conf
MACHINE = "qemux86-64"
DISTRO = "poky-tiny"
EOF
```

Now I want to save this setup so that it is reproducible.

I can save my layers and their remote's with the following command:

```sh
# save layer remotes
bitbake-layers create-layers-setup ../src/meta-vader
# save bblayers.conf and local.conf as a template
bitbake-layers save-build-conf ../src/meta-vader vader
```

```sh
# save template and conf files in layer
git -C ../src/meta-vader add .
git -C ../src/meta-vader commit -m "save setup-layers and template"
```

Now I want to reproduce this setup on another machine:

```sh
# exit docker <C-d>

# 1. clone bootstrap Yocto layer
git clone ~/yocto-project/src/meta-vader/ ~/yocto-project2/src/meta-vader
cd ~/yocto-project2/
```

Copy over docker config files again:

```sh
cp -r ~/src/yocto-tooling/.cqfd* .
cqfd init
cqfd shell
```

Now that I have cloned my bootstrap layer to the location I want I can setup
all the other layers:

```sh
# 2. run setup-layers to clone other layers
./src/meta-vader/setup-layers
```

This clones all the layers to the same directory as `meta-vader`:

Now `setup-build` or the classic `oe-init-build-env`

```sh
# 3. setup build env and conf files
./src/setup-build setup -c vader-vader -b build
```

or

```sh
# 3. setup build env and conf files
TEMPLATECONF=$PWD/src/meta-vader/conf/templates/vader source ./src/poky/oe-init-build-env
```

```sh
# 4. build
bitbake -k core-image-minimal
```

Personally not a big fan because it requires more steps which are prone to
human error.

Where as with kas the process would look like:

```sh
# 1. clone project
git clone <url> yocto-project
cd yocto-project
# 2. build
kas build
```

Plus you don't have to manually clone the bootstrap layer into src. So with kas
the locations of the layers in relation to the root of the project are preserved
which is not the case with the official Yocto method. Which isn't an issue if
all your layers are at the root of the project but I prefer them in a src
folder so that it is easier to grep/find.

```sh
# grepping layers if they are in the root
grep -rn placeholder poky meta-openembedded meta-vader
# grepping layers if they are in src
grep -rn placeholder src
```

## bitbake-layers

Now so far I have shown you every bitbake-layers command except:

```sh
bitbake-layers show-recipes
bitbake-layers show-recipes -l meta-python
bitbake-layers show-recipes -i kernel
```

```sh
bitbake-layers flatten layer1 layer2 output-layer
# flatten layer configuration into a separate output directory.
```
Can be used to merge layers.

```sh
bitbake-layers show-overlayed
```
Shows when multiple layers provide the same recipe.

```sh
bitbake-layers show-appends
bitbake-layers show-appends linux-yocto
```

```sh
bitbake-layers show-cross-depends
```
Show dependencies between recipes that cross layer boundaries. Great for
determining what layers your layer depends on to correctly set
`LAYERDEPENDS_meta-<layer>` in layer.conf.

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

~/src/yocto-tooling/patches/0001-kernel.bbclass-generate-compile_commands.json.patch

## bitbake

Most used bitbake commands:

```sh
bitbake core-image-minimal
bitbake -k core-image-minimal
bitbake -n core-image-minimal
bitbake -p core-image-minimal
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
```

```sh
bitbake -g core-image-minimal
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
recipetool appendsrcfile ../src/meta-vader virtual/kernel this_defconfig arch/x86/configs/this_defconfig
# although your kernel provider may have their own way to implement this
```

# devtool

```sh
# creates this recipe in the workspace for you to edit
devtool add complex-script /tmp/example-source-code/complex.sh
devtool finish complex-script meta-vader
```

```sh
# when I want to patch the kernel
devtool modify virtual/kernel
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

<!-- ```sh -->
<!-- kin@f16059a05e95:~/yocto-project/build$ devtool -h -->
<!-- NOTE: Starting bitbake server... -->
<!-- usage: devtool [--basepath BASEPATH] [--bbpath BBPATH] [-d] [-q] [--color COLOR] [-h] <subcommand> ... -->
<!---->
<!-- OpenEmbedded development tool -->
<!---->
<!-- options: -->
<!--   --basepath BASEPATH   Base directory of SDK / build directory -->
<!--   --bbpath BBPATH       Explicitly specify the BBPATH, rather than getting it from the metadata -->
<!--   -d, --debug           Enable debug output -->
<!--   -q, --quiet           Print only errors -->
<!--   --color COLOR         Colorize output (where COLOR is auto, always, never) -->
<!--   -h, --help            show this help message and exit -->
<!---->
<!-- subcommands: -->
<!--   Beginning work on a recipe: -->
<!--     add                   Add a new recipe -->
<!--     modify                Modify the source for an existing recipe -->
<!--     upgrade               Upgrade an existing recipe -->
<!--   Getting information: -->
<!--     status                Show workspace status -->
<!--     latest-version        Report the latest version of an existing recipe -->
<!--     check-upgrade-status  Report upgradability for multiple (or all) recipes -->
<!--     search                Search available recipes -->
<!--   Working on a recipe in the workspace: -->
<!--     ide-sdk               Setup the SDK and configure the IDE -->
<!--     build                 Build a recipe -->
<!--     rename                Rename a recipe file in the workspace -->
<!--     edit-recipe           Edit a recipe file -->
<!--     find-recipe           Find a recipe file -->
<!--     configure-help        Get help on configure script options -->
<!--     update-recipe         Apply changes from external source tree to recipe -->
<!--     reset                 Remove a recipe from your workspace -->
<!--     finish                Finish working on a recipe in your workspace -->
<!--   Testing changes on target: -->
<!--     deploy-target         Deploy recipe output files to live target machine -->
<!--     undeploy-target       Undeploy recipe output files in live target machine -->
<!--     build-image           Build image including workspace recipe packages -->
<!--   Advanced: -->
<!--     create-workspace      Set up workspace in an alternative location -->
<!--     import                Import exported tar archive into workspace -->
<!--     extract               Extract the source for an existing recipe -->
<!--     sync                  Synchronize the source tree for an existing recipe -->
<!--     export                Export workspace into a tar archive -->
<!--     menuconfig            Alter build-time configuration for a recipe -->
<!-- Use devtool <subcommand> --help to get help on a specific command -->
<!-- ``` -->
