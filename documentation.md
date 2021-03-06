---
layout: default
---

Advanced usage
==============

## Build image ##
You can build a new JuNest image from scratch by running the following command:

    junest -b [-n]

The script will create a directory containing all the essentials
files in order to make JuNest working properly (such as pacman, yaourt and proot).
The option **-n** will skip the final validation tests if they are not needed.
Remember that the script to build the image must run in an Arch Linux OS with
arch-install-scripts, package-query, git and the base-devel packages installed.
To change the build directory just use the *JUNEST_TEMPDIR* (by default /tmp).

After creating the image junest-x86\_64.tar.gz you can install it by running:

    junest -i junest-x86_64.tar.gz

For more details, you can also take a look at
[junest-builder](https://github.com/fsquillace/junest-builder)
that contains the script and systemd service used for the automatic building
of the JuNest image.

Related wiki page:

- [How to build a JuNest image using QEMU](https://github.com/fsquillace/junest/wiki/How-to-build-a-JuNest-image-using-QEMU)

## Run JuNest using a different architecture via QEMU ##
The following command will download the ARM JuNest image and will run QEMU in
case the host OS runs on either x86\_64 or x86 architectures:

    $> JUNEST_HOME=~/.junest-arm junest -a arm -- uname -m
    armv7l

## Bind directories ##
To bind a host directory to a guest location, you can use proot arguments:

    junest -p "-b /mnt/mydata:/home/user/mydata"

Check out the proot options with:

    junest -p "--help"

## Systemd integration ##

Although JuNest has not been designed to be a complete container, it is even possible to
virtualize the process tree thanks to the [systemd container](https://wiki.archlinux.org/index.php/Systemd-nspawn).
The JuNest containter allows to run services inside the container that can be
visible from the host OS through the network.
The drawbacks of this are that the host OS must use systemd as a service manager,
and the container can only be executed using root privileges.

To boot a JuNest container:

    sudo systemd-nspawn -bD ~/.junest

Related wiki page:

- [How to run junest as a container](https://github.com/fsquillace/junest/wiki/How-to-run-JuNest-as-a-container)
- [How to run services using Systemd](https://github.com/fsquillace/junest/wiki/How-to-run-services-using-Systemd)

Internals
=========

There are two main chroot jail used in JuNest.
The main one is [proot](https://wiki.archlinux.org/index.php/Proot) which
allows unprivileged users to execute programs inside a sandbox and
jchroot, a small and portable version of
[arch-chroot](https://wiki.archlinux.org/index.php/Chroot) which is an
enhanced chroot for privileged users that mounts the primary directories
(i.e. /proc, /sys, /dev and /run) before executing any programs inside
the sandbox.

## Automatic fallback to classic chroot ##
If jchroot fails for some reasons in the host system (i.e. it is not able to
mount one of the directories),
JuNest automatically tries to fallback to the classic chroot.

## Automatic fallback for all the dependent host OS executables ##
JuNest attempt first to run the executables in the host OS located in different
positions (/usr/bin, /bin, /usr/sbin and /sbin).
As a fallback it tries to run the same executable if it is available in the JuNest
image.

## Automatic building of the JuNest images ##
The JuNest images are built every week so that you can always get the most
updated package versions.

## Static QEMU binaries ##
There are static QEMU binaries included in JuNest image that allows to run JuNest
in a different architecture from the host system. They are located in `/opt/qemu`
directory.

Troubleshooting
===============

## Cannot use AUR repository ##

> **Q**: Why do I get the following error when I try to install a package with yaourt?

    Cannot find the gzip binary required for compressing man and info pages.

> **A**: JuNest comes with a very basic number of packages.
> In order to install packages using yaourt you may need to install the package group **base-devel**
> that contains all the essential packages for compiling source code (such as gcc, make, patch, etc):

    #> pacman -S base-devel

## No servers configured for repository ##

> **Q**: Why I cannot install packages?

    #> pacman -S lsof
    Packages (1): lsof-4.88-2

    Total Download Size:    0.09 MiB
    Total Installed Size:   0.21 MiB

    error: no servers configured for repository: core
    error: no servers configured for repository: community
    error: failed to commit transaction (no servers configured for repository)
    Errors occurred, no packages were upgraded.

> **A**: You need simply to update the mirrorlist file according to your location:

    # Uncomment the repository line according to your location
    #> nano /etc/pacman.d/mirrorlist
    #> pacman -Syy

## Locate the package for a given file ##

> **Q**: How do I find which package a certain file belongs to?

> **A**: JuNest is a really small distro, therefore you frequently need to find
> the package name for a certain file. `pkgfile` is an extremely useful package
> that allows you to detect the package of a given file.
> For instance, if you want to find the package name for the command `getopt`:

    #> pacman -S pkgfile
    #> pkgfile --update
    $> pkgfile getop
    core/util-linux

## Kernel too old ##

> **Q**: Why do I get the error: "FATAL: kernel too old"?

> **A**: This is because the executable from the precompiled package cannot
> properly run if the kernel is old.
> You may need to specify the PRoot *-k* option if the guest rootfs
> requires a newer kernel version:

    $> junest -p "-k 3.10"

> In order to check if an executable inside JuNest environment can be compatible
> with the kernel of the host OS just use the *file* command, for instance:

    $> file ~/.junest/usr/bin/bash
    ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked
    (uses shared libs), for GNU/Linux 2.6.32,
    BuildID[sha1]=ec37e49e7188ff4030052783e61b859113e18ca6, stripped

> From the output you can see what is the minimum recommended Linux kernel version.

## SUID permissions ##
> **Q**: Why I do not have permissions for ping?

    $> ping www.google.com
    ping: icmp open socket: Operation not permitted

> **A**: The ping command uses *suid* permissions that allow to execute the command using
> root privileges. The fakeroot mode is not able to execute a command set with suid,
> and you may need to use root privileges. There are other few commands that
> have *suid* permission, you can list the commands from your JuNest environment
> with the following command:

    $> find /usr/bin -perm +4000

## No characters are visible on a graphic application ##

> **Q**: Why I do not see any characters in the application I have installed?

> **A**: This is probably because there are no
> [fonts](https://wiki.archlinux.org/index.php/Font_Configuration) installed in
> the system.

> To quick fix this, you can just install a fonts package:

    #> pacman -S gnu-free-fonts

## Differences between filesystem and package ownership ##

> **Q**: Why do I get warning when I install a package using root privileges?

    #> pacman -S systat
    ...
    warning: directory ownership differs on /usr/
    filesystem: 1000:100  package: 0:0
    ...

> **A**: In these cases the package installation went smoothly anyway.
> This should happen every time you install package with root privileges
> since JuNest will try to preserve the JuNest environment by assigning ownership
> of the files to the real user.

More documentation
==================
There are additional tutorials in the
[JuNest wiki page](https://github.com/fsquillace/junest/wiki).
