# yocto-tooling

To begin I want to show how to setup Yocto using standard yocto tooling (ie not
using kas or repo).

<!-- |                              | combo-layer | oe-setup-layers | templates | git submodules | repo | kas | kas-container | -->
<!-- |------------------------------|-------------|-----------------|-----------|----------------|------|-----|---------------| -->
<!-- | multi repo management        |      x      |        x        |           |       x        |  x   |  x  |      x        | -->
<!-- | build environment management |             |        x        |     x     |                |      |  x  |      x        | -->
<!-- | layer management             |             |        x        |     x     |                |      |  x  |      x        | -->
<!-- | container management         |             |                 |           |                |      |     |      x        | -->
<!-- | patch management             |      x      |                 |           |                |      |  x  |      x        | -->

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
cd
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
bitbake-layers layerindex-fetch --shallow --fetchdir ../src meta-python
bitbake-layers show-layers
```

Now lets add a custom layer but we need to track it with git so that
setup-layers treats it as a layer:

```sh
bitbake-layers create-layer ../src/meta-vader
git -C ../src/meta-vader init
git -C ../src/meta-vader add .
git -C ../src/meta-vader commit -m "initial commit"
git -C ../src/meta-vader remote add origin ~/yocto-project/src/meta-vader
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
# save template and conf files in layer
git -C ../src/meta-vader add .
git -C ../src/meta-vader commit -m "save setup-layers and template"
```

Now I want to reproduce this setup on another machine:

```sh
# exit docker <C-d>
cd
mkdir ~/yocto-project2
cd ~/yocto-project2
git clone ~/yocto-project/src/meta-vader/ ./src/meta-vader
```

Now that I have cloned my bootstrap layer to the location I want I can setup
all the other layers:

```sh
./src/meta-vader/setup-layers
```

This clones all the layers to the same directory as `meta-vader`:

Copy over docker config files again:

```sh
cp -r ~/src/yocto-tooling/.cqfd* .
cqfd init
cqfd shell
```

Now `setup-build` or the classic `oe-init-build-env`

```sh
./src/setup-build setup -c vader-vader -b build
```

or

```sh
TEMPLATECONF=$PWD/src/meta-vader/conf/templates/vader source ./src/poky/oe-init-build-env
```

# Topics to touch on

- bitbake
- bitbake-getvar
- makefile-getvar
- bitbake-layers
- bitbake-config-build
- bitbake-diffsigs/bitbake-dumpsig
- bblock
- oe-setup-layers
- devtool
- recipetool
- runqemu
- oe-depends-dot
- oe-pkgdata-util
- oe-run-native

# Notes

- autobuilder-worker-prereq-tests
    For CI to check for build prerequisits

- b4-wrapper-poky.py
    script to be called from b4

- bblock
    lock recipes so that they are not rebuilt
    bblock.rst

- bitbake
    -g
    -e
    -u
    -f
    -c
    --runall
    --runonly
    -n
    -k
    -p
    -P
    -D
    -w

    Cool, look into server options

- bitbake-config-build
    used to specify build configuration fragments
    for like setting up CI or other fragments of options

    ```sh
    find -path \*conf/fragments/\*/\*.conf
    ```

- bitbake-diffsigs/bitbake-dumpsig
- bitbake-hashclient
- bitbake-hashserv
- bitbake-prserv
- bitbake-prserv-tool
- clean-hashserver-database

- bitbake-getvar
    better alternative to `bitbake -e | grep` since it shows the history

- bitbake-layers
    PRIORITY tool

- bitbake-selftest
    running bitbake tests

- bitbake-server
    should not be ran by users

- bitbake-worker
    should not be ran by users

- buildall-qemu
    tool for automating build testing of recipes

- buildhistory-collect-srcrevs
    -a can be used to lock SRCREVs used to override AUTOREV default settings
    watch out overrides can still take precedence if not used with -f

- buildhistory-diff
    Report significant differences in the buildhistory repository since a specific revision

- buildstats-diff
    Script for comparing buildstats from two different builds

- buildstats-summary
    Dump a summary of the specified buildstats to the terminal, filtering and
    sorting by walltime.

- combo-layer
    Yocto specific version of repo or submodules
    - combo-layer.conf.example
    - combo-layer-hook-default.sh

- cp-noerror
    cp with no error if files disappear

- create-pull-request
    used for contributing, preparing patches and cover letters

- crosstap
    sets up stuff for systemtap scripts
    linux diagnostics tooling

- cve-json-to-text.py
    exactly what it sounds like

- devtool
    PRIORITY tool

- gen-lockedsig-cache
    idk

- git-make-shallow
    idk

- install-buildtools
    script for installing build tools dependencies
    could be cool to use in Dockerfiles instead of all of them explicitly

- lz4c
    not too useful for you currently

- makefile-getvar
    cool may be good to mention

- oe-buildenv-internal
    used internally by oe-init-build-env

- oe-build-perf-test
- oe-build-perf-report
    generate build performance reports

- oe-check-sstate
- oe-debuginfod

- oe-depends-dot
    so useful for determining why recipes get included

- oe-find-native-sysroot
    This script is intended to be run within other scripts by source'ing

- oe-git-archive
- oe-git-proxy
    idk im not going to dive into these

- oe-gnome-terminal-phonehome
    idk

- oe-pkgdata-browser
- oe-pkgdata-browser.glade
    gui for browsing pkgdata

- oe-pkgdata-util
    PRIORITY tool
    super useful

- oe-publish-sdk
    idk

- oepydevshell-internal.py
    getting a pydevshell

- oe-pylint
    pylint wrapper

- oe-run-native
    PRIORITY tool
    run native tools

- oe-selftest
    runs testsuites for oe stuff

- oe-setup-layers
    - calls oe-setup-build
        sets up local.conf and bblayers.conf

- oe-init-build-env
    - sets up OEROOT
    - oe-buildenv-internal
        - sets up build environment
    - oe-setup-builddir
        sources `$OEROOT/.templateconf`
        you can use .templateconf to select templates for local.conf and
        bblayers.conf

        populates the local.conf and bblayers.conf from the default templates

        ```sh
        . "$OEROOT/.templateconf"
        ...
        OECORELAYERCONF="$TEMPLATECONF/bblayers.conf.sample"
        OECORELOCALCONF="$TEMPLATECONF/local.conf.sample"
        OECORESUMMARYCONF="$TEMPLATECONF/conf-summary.txt"
        OECORENOTESCONF="$TEMPLATECONF/conf-notes.txt"
        ```
        ```sh
        # from .templateconf
        TEMPLATECONF=${TEMPLATECONF:-meta-poky/conf/templates/default}
        ```

    - oe-setup-vscode
        - sets up vscode config files

- oe-test
    for testing openembedded

- oe-time-dd-test.sh
    test what part of the build system stresses the filesystem

- oe-trim-schemas
    gnome configuration simplification script

- opkg-query-helper.py
    idk

- patchtest
- patchtest-get-branch
- patchtest-get-series
- patchtest.README
- patchtest-send-results
- patchtest-setup-sharedir
    for testing patches for contributions

- pull-sdpx-licenses.py
    idk

- pythondeps
    used by recipetool

- recipetool
    used for creating recipes n stuff
    TODO: deep dive

- relocate_sdk.py
    relocates sdk
    used by the sdk installer script

- resulttool
    test results tool - tool for manipulating OEQA test result json files

- rootfs_rpm-extract-postinst.awk
    10 line awk script for filtering specific lines in postinsts scripts

- rpm2cpio.sh

- runqemu
- runqemu-addptable2image
- runqemu-export-rootfs
- runqemu-extract-sdk
- runqemu-gen-tapdevs
- runqemu-ifdown
- runqemu-ifup
- runqemu.README

- send-error-report
- send-pull-request
- sstate-cache-management.py
- sstate-diff-machines.sh
- sstate-sysroot-cruft.sh
- sysroot-relativelinks.py
- task-time
- test-reexec
- test-remote-image
- toaster
- toaster-eventreplay
- verify-bashisms
- wic
- yocto-check-layer
- yocto-check-layer-wrapper
- yocto_testresults_query.py
