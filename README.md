# yocto-tooling

To begin I want to show how to setup Yocto using stand yocto tooling (ie not
using kas or repo).

# First setup build environment

So first off when setting up a Yocto project you need some way to manage your
layers, which are made up from several repos. This can be done by tools like
git submodules, repo, or kas.

### combo-layer vs oe-setup-layers

The `poky` codebase itself is a multirepo project so it needs to manage its
repos with such a tool.

The tool of choice for managing multiple layers in Yocto is a hand-rolled tool
called combo-layer.

So firstly we will need to create a combo-layer configuration file.

    combo-layer.conf

Ironically the combo-layer tool is in the repo we want to specify in this
sample project so we will copy a temporary version of poky to get it:

```sh
git clone git://git.yoctoproject.org/poky --depth 1 /tmp/poky
/tmp/poky/scripts/combo-layer -c combo-layer.conf init
/tmp/poky/scripts/combo-layer -c combo-layer.conf update poky
rm -rf /tmp/poky
```

### build dependencies

Now that we have poky we should also make sure that we have a reproducible
build environment. Ideally dockerized so we will use `cqfd`. Note it is not a
Yocto tool it is just the best docker wrapper in my opinion.

- install-buildtools + cqfd

I tried to use the install-buildtools script inside docker but it would still
not install all the host dependencies once I finally got it working:

```sh
kin@28413da5c3a2:~/src/yocto-tooling/build$ bitbake -n core-image-minimal
ERROR: The following required tools (as specified by HOSTTOOLS) appear to be unavailable in PATH, please install them in order to proceed:
  bunzip2 bzip2 cpio diffstat gawk
```

Personally it just seems easier to use an ubuntu dockerfile and install all the
dependencies listed in the Yocto docs. This install-buildtools method requires
you to source an extra script as well which can be easily forgotten.

- bitbake-layers create-layers-setup <destdir> + oe-setup-layers + setup-build
    doesn't retain the location of the layers properly IMO

- oe-init-build-env && .templateconf

# Topics to touch on

- bitbake
- bitbake-getvar
- makefile-getvar
- bitbake-layers
- bitbake-config-build
- bitbake-diffsigs/bitbake-dumpsig
- bblock
- combo-layer (use instead of submodules)
- oe-setup-layers
- devtool
- recipetool
- runqemu
- oe-depends-dot
- oe-pkgdata-util
- oe-run-native
- install-buildtools + cqfd

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
