=======
 PRoot
=======

-------------------------------------------------------------------------
``chroot``, ``mount --bind``, and ``binfmt_misc`` without privilege/setup
-------------------------------------------------------------------------

:Date: 2021-09-01
:Version: 5.2.0
:Manual section: 1


Synopsis
========

**proot** [*option*] ... [*command*]


Description
===========

PRoot is a user-space implementation of ``chroot``, ``mount --bind``,
and ``binfmt_misc``.  This means that users don't need any privileges
or setup to do things like using an arbitrary directory as the new
root filesystem, making files accessible somewhere else in the
filesystem hierarchy, or executing programs built for another CPU
architecture transparently through QEMU user-mode.  Also, developers
can use PRoot as a generic Linux process instrumentation engine thanks
to its extension mechanism, see CARE_ for an example.  Technically
PRoot relies on ``ptrace``, an unprivileged system-call available in
every Linux kernel.

The new root file-system, a.k.a *guest rootfs*, typically contains a
Linux distribution.  By default PRoot confines the execution of
programs to the guest rootfs only, however users can use the built-in
*mount/bind* mechanism to access files and directories from the actual
root file-system, a.k.a *host rootfs*, just as if they were part of
the guest rootfs.

When the guest Linux distribution is made for a CPU architecture
incompatible with the host one, PRoot uses the CPU emulator QEMU
user-mode to execute transparently guest programs.  It's a convenient
way to develop, to build, and to validate any guest Linux packages
seamlessly on users' computer, just as if they were in a *native*
guest environment.  That way all of the cross-compilation issues are
avoided.

PRoot can also *mix* the execution of host programs and the execution
of guest programs emulated by QEMU user-mode.  This is useful to use
host equivalents of programs that are missing from the guest rootfs
and to speed up build-time by using cross-compilation tools or
CPU-independent programs, like interpreters.

It is worth noting that the guest kernel is never involved, regardless
of whether QEMU user-mode is used or not.  Technically, when guest
programs perform access to system resources, PRoot translates their
requests before sending them to the host kernel.  This means that
guest programs can use host resources (devices, network, ...) just as
if they were "normal" host programs.

.. _CARE: https://proot-me.github.io/care


Options
=======

The command-line interface is composed of two parts: first PRoot's
options (optional), then the command to launch (``/bin/sh`` if not
specified).  This section describes the options supported by PRoot,
that is, the first part of its command-line interface.


Regular options
---------------

-r path, --rootfs=path
    Use *path* as the new guest root file-system, default is ``/``.

    The specified *path* typically contains a Linux distribution where
    all new programs will be confined.  The default rootfs is ``/``
    when none is specified, this makes sense when the bind mechanism
    is used to relocate host files and directories, see the ``-b``
    option and the ``Examples`` section for details.

    It is recommended to use the ``-R`` or ``-S`` options instead.

-b path, --bind=path, -m path, --mount=path
    Make the content of *path* accessible in the guest rootfs.

    This option makes any file or directory of the host rootfs
    accessible in the confined environment just as if it were part of
    the guest rootfs.  By default the host path is bound to the same
    path in the guest rootfs but users can specify any other location
    with the syntax: ``-b *host_path*:*guest_location*``.  If the
    guest location is a symbolic link, it is dereferenced to ensure
    the new content is accessible through all the symbolic links that
    point to the overlaid content.  In most cases this default
    behavior shouldn't be a problem, although it is possible to
    explicitly not dereference the guest location by appending it the
    ``!`` character: ``-b *host_path*:*guest_location!*``.

-q command, --qemu=command
    Execute guest programs through QEMU as specified by *command*.

    Each time a guest program is going to be executed, PRoot inserts
    the QEMU user-mode *command* in front of the initial request.
    That way, guest programs actually run on a virtual guest CPU
    emulated by QEMU user-mode.  The native execution of host programs
    is still effective and the whole host rootfs is bound to
    ``/host-rootfs`` in the guest environment.

-w path, --pwd=path, --cwd=path
    Set the initial working directory to *path*.

    Some programs expect to be launched from a given directory but do
    not perform any ``chdir`` by themselves.  This option avoids the
    need for running a shell and then entering the directory manually.

-v value, --verbose=value
    Set the level of debug information to *value*.

    The higher the integer *value* is, the more detailed debug
    information is printed to the standard error stream.  A negative
    *value* makes PRoot quiet except on fatal errors.

-V, --version, --about
    Print version, copyright, license and contact, then exit.

-h, --help, --usage
    Print the version and the command-line usage, then exit.


Extension options
-----------------

The following options enable built-in extensions.  Technically
developers can add their own features to PRoot or use it as a Linux
process instrumentation engine thanks to its extension mechanism, see
the sources for further details.

-k string, --kernel-release=string
    Make current kernel appear as kernel release *string*.

    If a program is run on a kernel older than the one expected by its
    GNU C library, the following error is reported: "FATAL: kernel too
    old".  To be able to run such programs, PRoot can emulate some of
    the features that are available in the kernel release specified by
    *string* but that are missing in the current kernel.

-0, --root-id
    Make current user appear as "root" and fake its privileges.

    Some programs will refuse to work if they are not run with "root"
    privileges, even if there is no technical reason for that.  This
    is typically the case with package managers.  This option allows
    users to bypass this kind of limitation by faking the user/group
    identity, and by faking the success of some operations like
    changing the ownership of files, changing the root directory to
    ``/``, ...  Note that this option is quite limited compared to
    ``fakeroot``.

-i string, --change-id=string
    Make current user and group appear as *string* "uid:gid".

    This option makes the current user and group appear as *uid* and
    *gid*.  Likewise, files actually owned by the current user and
    group appear as if they were owned by *uid* and *gid* instead.
    Note that the ``-0`` option is the same as ``-i 0:0``.

-p string, --port=string
    Map ports to others with the syntax as *string* "port_in:port_out ...".

    This option makes PRoot intercept bind and connect system calls,
    and change the port they use. The port map is specified
    with the syntax: ``-b *port_in*:*port_out*``. For example,
    an application that runs a MySQL server binding to 5432 wants
    to cohabit with other similar application, but doesn't have an
    option to change its port. PRoot can be used here to modify
    this port: ``proot -p 5432:5433 myapplication``. With this command,
    the MySQL server will be bound to the port 5433.
    This command can be repeated multiple times to map multiple ports.

-n, --netcoop
    Activates the network cooperation mode.

    This option makes PRoot intercept bind() system calls and
    change the port they are binding to to 0. With this, the system will
    allocate an available port. Each time this is done, a new entry is added
    to the port mapping entries, so that corresponding connect() system calls
    use the same resulting port.

Alias options
-------------

The following options are aliases for handy sets of options.

-R path
    Alias: ``-r *path*`` + a couple of recommended ``-b``.

    Programs isolated in *path*, a guest rootfs, might still need to
    access information about the host system, as it is illustrated in
    the ``Examples`` section of the manual.  These host information
    are typically: user/group definition, network setup, run-time
    information, users' files, ...  On all Linux distributions, they
    all lie in a couple of host files and directories that are
    automatically bound by this option:

    * /etc/host.conf
    * /etc/hosts
    * /etc/hosts.equiv
    * /etc/mtab
    * /etc/netgroup
    * /etc/networks
    * /etc/passwd
    * /etc/group
    * /etc/nsswitch.conf
    * /etc/resolv.conf
    * /etc/localtime
    * /dev/
    * /sys/
    * /proc/
    * /tmp/
    * /run/
    * /var/run/dbus/system_bus_socket
    * $HOME
    * *path*

-S path
    Alias: ``-0 -r *path*`` + a couple of recommended ``-b``.

    This option is useful to safely create and install packages into
    the guest rootfs.  It is similar to the ``-R`` option except it
    enables the ``-0`` option and binds only the following minimal set
    of paths to avoid unexpected changes on host files:

    * /etc/host.conf
    * /etc/hosts
    * /etc/nsswitch.conf
    * /etc/resolv.conf
    * /dev/
    * /sys/
    * /proc/
    * /tmp/
    * /run/shm
    * $HOME
    * *path*


Exit Status
===========

If an internal error occurs, ``proot`` returns a non-zero exit status,
otherwise it returns the exit status of the last terminated
program. When an error has occurred, the only way to know if it comes
from the last terminated program or from ``proot`` itself is to have a
look at the error message.


Files
=====

PRoot reads links in ``/proc/<pid>/fd/`` to support `openat(2)`-like
syscalls made by the guest programs.


Examples
========

In the following examples the directories ``/mnt/slackware-8.0`` and
``/mnt/armslack-12.2/`` contain a Linux distribution respectively made
for x86 CPUs and ARM CPUs.


``chroot`` equivalent
---------------------

To execute a command inside a given Linux distribution, just give
``proot`` the path to the guest rootfs followed by the desired
command.  The example below executes the program ``cat`` to print the
content of a file::

    proot -r /mnt/slackware-8.0/ cat /etc/motd
    
    Welcome to Slackware Linux 8.0

The default command is ``/bin/sh`` when none is specified. Thus the
shortest way to confine an interactive shell and all its sub-programs
is::

    proot -r /mnt/slackware-8.0/
    
    $ cat /etc/motd
    Welcome to Slackware Linux 8.0


``mount --bind`` equivalent
---------------------------

The bind mechanism enables one to relocate files and directories.  This is
typically useful to trick programs that perform access to hard-coded
locations, like some installation scripts::

    proot -b /tmp/alternate_opt:/opt
    
    $ cd to/sources
    $ make install
    [...]
    install -m 755 prog "/opt/bin"
    [...] # prog is installed in "/tmp/alternate_opt/bin" actually

As shown in this example, it is possible to bind over files not even
owned by the user.  This can be used to *overlay* system configuration
files, for instance the DNS setting::

    ls -l /etc/hosts
    -rw-r--r-- 1 root root 675 Mar  4  2011 /etc/hosts

::

    proot -b ~/alternate_hosts:/etc/hosts
    
    $ echo '1.2.3.4 google.com' > /etc/hosts
    $ resolveip google.com
    IP address of google.com is 1.2.3.4
    $ echo '5.6.7.8 google.com' > /etc/hosts
    $ resolveip google.com
    IP address of google.com is 5.6.7.8

Another example: on most Linux distributions ``/bin/sh`` is a symbolic
link to ``/bin/bash``, whereas it points to ``/bin/dash`` on Debian
and Ubuntu.  As a consequence a ``#!/bin/sh`` script tested with Bash
might not work with Dash.  In this case, the binding mechanism of
PRoot can be used to set non-disruptively ``/bin/bash`` as the default
``/bin/sh`` on these two Linux distributions::

    proot -b /bin/bash:/bin/sh [...]

Because ``/bin/sh`` is initially a symbolic link to ``/bin/dash``, the
content of ``/bin/bash`` is actually bound over this latter::

    proot -b /bin/bash:/bin/sh
    
    $ md5sum /bin/sh
    089ed56cd74e63f461bef0fdfc2d159a  /bin/sh
    $ md5sum /bin/bash
    089ed56cd74e63f461bef0fdfc2d159a  /bin/bash
    $ md5sum /bin/dash
    089ed56cd74e63f461bef0fdfc2d159a  /bin/dash

In most cases this shouldn't be a problem, but it is still possible to
strictly bind ``/bin/bash`` over ``/bin/sh`` -- without dereferencing
it -- by specifying the ``!`` character at the end::

    proot -b '/bin/bash:/bin/sh!'
    
    $ md5sum /bin/sh
    089ed56cd74e63f461bef0fdfc2d159a  /bin/sh
    $ md5sum /bin/bash
    089ed56cd74e63f461bef0fdfc2d159a  /bin/bash
    $ md5sum /bin/dash
    c229085928dc19e8d9bd29fe88268504  /bin/dash


``chroot`` + ``mount --bind`` equivalent
----------------------------------------

The two features above can be combined to make any file from the host
rootfs accessible in the confined environment just as if it were
initially part of the guest rootfs.  It is sometimes required to run
programs that rely on some specific files::

    proot -r /mnt/slackware-8.0/
    
    $ ps -o tty,command
    Error, do this: mount -t proc none /proc

works better with::

    proot -r /mnt/slackware-8.0/ -b /proc
    
    $ ps -o tty,command
    TT       COMMAND
    ?        bash
    ?        proot -b /proc /mnt/slackware-8.0/
    ?        sh
    ?        ps -o tty,command

Actually there's a bunch of such specific files, that's why PRoot
provides the option ``-R`` to bind automatically a pre-defined list of
recommended paths::

    proot -R /mnt/slackware-8.0/
    
    $ ps -o tty,command
    TT       COMMAND
    pts/6    bash
    pts/6    proot -R /mnt/slackware-8.0/
    pts/6    sh
    pts/6    ps -o tty,command


``chroot`` + ``mount --bind`` + ``su`` equivalent
-------------------------------------------------

Some programs will not work correctly if they are not run by the
"root" user, this is typically the case with package managers.  PRoot
can fake the root identity and its privileges when the ``-0`` (zero)
option is specified::

    proot -r /mnt/slackware-8.0/ -0
    
    # id
    uid=0(root) gid=0(root) [...]
    
    # mkdir /tmp/foo
    # chmod a-rwx /tmp/foo
    # echo 'I bypass file-system permissions.' > /tmp/foo/bar
    # cat /tmp/foo/bar
    I bypass file-system permissions.

This option is typically required to create or install packages into
the guest rootfs.  Note it is *not* recommended to use the ``-R``
option when installing packages since they may try to update bound
system files, like ``/etc/group``.  Instead, it is recommended to use
the ``-S`` option.  This latter enables the ``-0`` option and binds
only paths that are known to not be updated by packages::

    proot -S /mnt/slackware-8.0/
    
    # installpkg perl.tgz
    Installing package perl...


``chroot`` + ``mount --bind`` + ``binfmt_misc`` equivalent
----------------------------------------------------------

PRoot uses QEMU user-mode to execute programs built for a CPU
architecture incompatible with the host one.  From users'
point-of-view, guest programs handled by QEMU user-mode are executed
transparently, that is, just like host programs.  To enable this
feature users just have to specify which instance of QEMU user-mode
they want to use with the option ``-q``::

    proot -R /mnt/armslack-12.2/ -q qemu-arm
    
    $ cat /etc/motd
    Welcome to ARMedSlack Linux 12.2

The parameter of the ``-q`` option is actually a whole QEMU user-mode
command, for instance to enable its GDB server on port 1234::

    proot -R /mnt/armslack-12.2/ -q "qemu-arm -g 1234" emacs

PRoot allows one to mix transparently the emulated execution of guest
programs and the native execution of host programs in the same
file-system namespace.  It's typically useful to extend the list of
available programs and to speed up build-time significantly.  This
mixed-execution feature is enabled by default when using QEMU
user-mode, and the content of the host rootfs is made accessible
through ``/host-rootfs``::

    proot -R /mnt/armslack-12.2/ -q qemu-arm
    
    $ file /bin/echo
    [...] ELF 32-bit LSB executable, ARM [...]
    $ /bin/echo 'Hello world!'
    Hello world!

    $ file /host-rootfs/bin/echo
    [...] ELF 64-bit LSB executable, x86-64 [...]
    $ /host-rootfs/bin/echo 'Hello mixed world!'
    Hello mixed world!

Since both host and guest programs use the guest rootfs as ``/``,
users may want to deactivate explicitly cross-filesystem support found
in most GNU cross-compilation tools.  For example with GCC configured
to cross-compile to the ARM target::

    proot -R /mnt/armslack-12.2/ -q qemu-arm
    
    $ export CC=/host-rootfs/opt/cross-tools/arm-linux/bin/gcc
    $ export CFLAGS="--sysroot=/"   # could be optional indeed
    $ ./configure; make

As with regular files, a host instance of a program can be bound over
its guest instance.  Here is an example where the guest binary of
``make`` is overlaid by the host one::

   proot -R /mnt/armslack-12.2/ -q qemu-arm -b /usr/bin/make
   
   $ which make
   /usr/bin/make
   $ make --version # overlaid
   GNU Make 3.82
   Built for x86_64-slackware-linux-gnu

It's worth mentioning that even when mixing the native execution of
host programs and the emulated execution of guest programs, they still
believe they are running in a native guest environment.  As a
demonstration, here is a partial output of a typical ``./configure``
script::

    checking whether the C compiler is a cross-compiler... no


Downloads
=========

PRoot
-----

The source code for PRoot and CARE are hosted in the same repository on `GitHub <https://github.com/proot-me/proot>`_.
Previous PRoot releases were packaged at https://github.com/proot-me/proot-static-build/releases, however, that
repository has since been archived. The latest builds can be found under the job artifacts for the `GitLab CI/CD Pipelines <https://gitlab.com/proot/proot/pipelines>`_ for each commit. The following commands can be used to download the latest x86_64 binary for convenience::

    curl -LO https://proot.gitlab.io/proot/bin/proot
    chmod +x ./proot
    proot --version

Rootfs
------

Here follows a couple of URLs where some rootfs archives can be freely
downloaded.  Note that ``mknod`` errors reported by ``tar`` when
extracting these archives can be safely ignored since special files
are typically bound (see ``-R`` option for details).

* https://download.openvz.org/template/precreated

* https://images.linuxcontainers.org/images

* http://distfiles.gentoo.org/releases

* http://cdimage.ubuntu.com/ubuntu-core

* https://archlinuxarm.org/about/downloads

* https://alpinelinux.org/downloads

Technically such rootfs archive can be created by running the
following command on the expected Linux distribution::

    tar --one-file-system --create --gzip --file my_rootfs.tar.gz /


Ecosystem
=========

The following ecosystem has developed around PRoot since it has been
made publicly available.

Projects using PRoot or CARE
----------------------------

* `ATOS
  <http://compilfr.ens-lyon.fr/wp-content/uploads/2013/12/17-Francois_DeFerriere.pdf>`_:
  find automatically C/C++ compiler options that provide best
  optimizations.

* CARE_: archive material used during an execution to make it
  reproducible on any Linux system.

* `Debian noroot
  <https://play.google.com/store/apps/details?id=com.cuntubuntu>`_:
  use Debian Linux on Android without root access.

* `GNURoot
  <https://play.google.com/store/apps/details?id=champion.gnuroot>`_:
  use several Linux distros on Android without root access.

* `JuNest <http://fsquillace.github.io/junest-site>`_:
  use Arch Linux on any Linux distros without root access.

* `OPAM2Debian <https://forge.ocamlcore.org/projects/opam2debian>`_:
  create Debian packages which contains a fully compiled OPAM
  installation.

* `OpenMOLE <https://www.openmole.org>`_:
  execute programs on distributed computing environments.

* `Polysquare Travis Container
  <https://github.com/polysquare/polysquare-travis-container>`_:
  use several Linux distros on Travis-CI without root access.

* `Portable PyPy <https://github.com/squeaky-pl/portable-pypy>`_:
  portable 32 and 64 bit x86 PyPy binaries.

* `SIO Workers <http://sioworkers.readthedocs.org/en/latest>`_:
  batch long-term computations with Python.


Third party packages
--------------------

Binaries from the Downloads_ section are likely more up-to-date.

* `Alpine Linux <https://pkgs.alpinelinux.org/packages?name=proot>`_

* `Arch Linux <https://aur.archlinux.org/packages/proot>`_

* `Debian <https://packages.debian.org/sid/proot>`_

* `Gentoo <http://packages.gentoo.org/package/sys-apps/proot>`_

* `NixOS <https://github.com/NixOS/nixpkgs/tree/master/pkgs/tools/system/proot>`_

* `Termux <https://wiki.termux.com/wiki/PRoot>`_

* `Ubuntu <https://launchpad.net/ubuntu/+source/proot>`_

* `University of Chicago RCC <https://rcc.uchicago.edu/docs/software/modules/proot/midway2/current.html>`_

* `Void Linux <https://github.com/void-linux/void-packages/tree/master/srcpkgs/proot>`_


Public material about PRoot or CARE
-----------------------------------

* articles on `Rémi's blog
  <https://blog.duraffort.fr/tag/proot.html>`_.  Rémi (a.k.a Ivoire)
  is one of the PRoot developers.

* presentation "`Software engineering tools based on syscall
  instrumentation
  <https://archive.fosdem.org/2014/schedule/event/syscall>`_" during
  FOSDEM 2014.

* presentation "`SW testing & Reproducing a LAVA failures locally
  using CARE <https://connect.linaro.org/resources/lcu14/lcu14-211-lava-use-cases-sw-testing-reproducing-a-lava-failures-locally-using-care>`_"
  during Linaro Connect USA 2014

* presentation and essay "`CARE: the Comprehensive Archiver for
  Reproducible Execution
  <http://c-mind.org/events/trust2014/presentations/trust14_care.pdf>`_"
  (`essay <http://dl.acm.org/citation.cfm?doid=2618137.2618138>`_)
  during TRUST 2014

* presentation "`An Introduction to the CARE tool (dead link)
  <#>`_"
  during HiPEAC CSW 2013

* presentation and essay "`PRoot: a Step Forward for QEMU User-Mode
  <http://adt.cs.upb.de/quf/quf11/quf2011_13.pdf>`_" (`proceedings
  <http://adt.cs.upb.de/quf/quf2011_proceedings.pdf>`_) during
  QUF'11

* tutorial "`How to install nix in home (on another distribution)
  <https://nixos.wiki/wiki/Nix_Installation_Guide#PRoot>`_"


Companies using PRoot or CARE internally
----------------------------------------

* STMicroelectronics
* Sony
* Ericsson
* Cisco
* Gogo
* Infinite Omicron, LLC.


See Also
========

chroot(1), mount(8), binfmt_misc, ptrace(2), qemu(1), sb2(1),
bindfs(1), fakeroot(1), fakechroot(1)


Colophon
========

Visit https://proot-me.github.io for help, bug reports, suggestions, patches, ...
Copyright (C) 2021 PRoot Developers, licensed under GPL v2 or later.

::

     _____ _____              ___
    |  __ \  __ \_____  _____|   |_
    |   __/     /  _  \/  _  \    _|
    |__|  |__|__\_____/\_____/\____|

