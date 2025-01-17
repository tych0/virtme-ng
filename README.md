![Alt text](/img/screenshot.png?raw=true "virtme-ng screenshot")

What is virtme-ng?
====================

virtme-ng is a tool that allows to easily and quickly recompile and test a
Linux kernel, starting from the source code.

It allows to recompile the kernel in few minutes (rather than hours), then the
kernel is automatically started in a virtualized environment that is an exact
copy-on-write copy of your live system, which means that any changes made to
the virtualized environment do not affect the host system.

In order to do this a minimal config is produced (with the bare minimum support
to test the kernel inside qemu), then the selected kernel is automatically
built and started inside qemu, using the filesystem of the host as a
copy-on-write snapshot.

This means that you can safely destroy the entire filesystem, crash the kernel,
etc. without affecting the host.

Kernels produced with virtme-ng are lacking lots of features, in order to
reduce the build time to the minimum and still provide you a usable kernel
capable of running your tests and experiments.

virtme-ng is based on virtme, written by Andy Lutomirski <luto@kernel.org>
([web][korg-web] | [git][korg-git]).

Quick start
===========

```
 $ uname -r
 5.19.0-23-generic
 $ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
 $ cd linux
 $ virtme-ng --commit v6.1-rc6
 ...
 $ uname -r
 6.1.0-rc6-virtme-ng
 ^
 |___ Now you have a shell inside a virtualized copy of your entire system,
      that is running the new kernel

 ("ctrl+a, x" to return back to your real system)
```

Installation
============

If you're running Ubuntu you can install virtme-ng from
ppa:arighi/virtme-ng:
```
 $ sudo add-apt-repository ppa:arighi/virtme-ng
 $ sudo apt install --yes virtme-ng
```

Alternatively, you can install via pip directly (after cloning this git
repository):
```
 $ pip3 install -r requirements.txt .
```

Requirements
============

 * You need Python 3.8 or higher

 * QEMU 1.6 or higher is recommended (QEMU 1.4 and 1.5 are partially supported
   using a rather ugly kludge)
   * You will have a much better experience if KVM is enabled.  That means that
     you should be on bare metal with hardware virtualization (VT-x or SVM)
     enabled or in a VM that supports nested virtualization.  On some Linux
     distributions, you may need to be a member of the "kvm" group.  Using
     VirtualBox or most VPS providers will fall back to emulation.

 * Depending on the options you use, you may need a statically linked `busybox`
   binary somewhere in your path.

 * You may need to install `crash` to use the memory dump inspection feature
   (see example below).

Examples
========

 - Build and run a kernel from a local git repository:
```
   $ virtme-ng
```

 - Build and run tag v6.1-rc3 a local kernel git repository:
```
   $ virtme-ng -c v6.1-rc3
```

 - Build and run a kernel 2 commits before the previously compiled kernel:
```
   $ virtme-ng --commit HEAD~2
```

 - Test the previously built kernel:
```
   $ virtme-ng -s
```

 - Only generate .config in the current kernel build directory:
```
   $ virtme-ng --kconfig
```

 - Test the currently running kernel in a safe snapshot of the system:
```
   $ virtme-ng -r
```

 - Test installed kernel 6.2.0-21-generic kernel in a safe snapshot of the
   system (NOTE: /boot/vmlinuz-6.2.0-21-generic needs to be accessible):
```
   $ virtme-ng -r 6.2.0-21-generic
```

 - Download and test kernel 6.2.0-1003-lowlatency from deb packages:
```
   $ mkdir test
   $ cd test
   $ apt download linux-image-6.2.0-1003-lowlatency linux-modules-6.2.0-1003-lowlatency
   $ for d in *.deb; do dpkg -x $d .; done
   $ virtme-ng -r ./boot/vmlinuz-6.2.0-1003-lowlatency
```

 - Generate and inspect a memory dump of the currently tested kernel (crash
   tool needs to be installed):
```
   $ virtme-ng -d
```

 - Save a memory dump of the running kernel to /tmp/vmcore.img
```
   $ virtme-ng -d --dump-file /tmp/vmcore.img
```

 - Test the tip of the latest kernel, building the kernel on a remote build
   host called "builder", running make inside a specific build chroot
   (managed remotely by schroot):
```
   $ virtme-ng --build-host builder \
     --build-host-exec-prefix "schroot -c chroot:kinetic-amd64 -- "
```

 - Run the previously compiled kernel and enable bridge networking (NOTE: you
   may get permission denied in some Ubuntu releases, I solved by doing a
   `sudo chmod u+s /usr/lib/qemu/qemu-bridge-helper`:
```
   $ virtme-ng -s --network bridge
```

 - Run the previously compiled kernel adding an additional virtio-scsi device:
```
   $ qemu-img create -f qcow2 /tmp/disk.img 8G
   $ virtme-ng -s --disk /tmp/disk.img
```

 - Recompile the kernel enabling Rust support (using specific versions of the
   Rust toolchain binaries):
```
   $ virtme-ng RUSTC=rustc-1.62 BINDGEN=bindgen-0.56 RUSTFMT=rustfmt-1.62
```

 - Build and test the arm64 kernel (using a separate chroot in
   /opt/chroot/arm64 as the main filesystem):
```
   $ virtme-ng --arch arm64 --root /opt/chroot/arm64/
```

 - Build upstream kernel 6.1-rc6, execute `uname -r` inside it and send the
   output to cowsay on the host:
```
   $ virtme-ng -c v6.1-rc6 --exec 'uname -r' 2>/dev/null | cowsay
    __________________
   < 6.1.0-rc6-virtme >
    ------------------
           \   ^__^
            \  (oo)\_______
               (__)\       )\/\
                   ||----w |
                   ||     ||
```

 - Run `uname -r` on the host, build upstream kernel v6.1-rc6, pass uname
   output to this kernel, run `uname -r` inside it, send the whole output to
   cowsay back on the host:
```
   $ uname -r | virtme-ng -c v6.1-rc6 --exec 'cat -; uname -r' 2>/dev/null | cowsay -n
    ___________________
   / 5.19.0-23-generic \
   \ 6.1.0-rc6-virtme  /
    -------------------
           \   ^__^
            \  (oo)\_______
               (__)\       )\/\
                   ||----w |
                   ||     ||
```

Implementation details
======================

virtme-ng allows to automatically configure, build and run kernels using the
main command-line interface called virtme-ng.

Initially a minimal custom .config is generated using (a custom version of)
virtme-configkernel.

It is possible to specify a set of custom configs (.config chunk) in
~/.config/virtme-ng/kernel.config, these user-specific settings will override
the default settings of virtme-configkernel (except for the mandatory configs
that are required to boot and test the kernel inside qemu, using virtme-run).

Then the kernel is compiled either locally or on an external build host (if the
`--build-host` option is used); once the build is done only the required files
needed to test the kernel are copied from the remote host if an external build
host is used.

When a remote build host is used (--build-host) the target branch is force
pushed to the remote host inside the ~/.virtme folder.

Then the kernel is executed using the virtme module. This allows to test the
kernel using a safe copy-on-write snapshot of the entire host filesystem.

All the kernels compiled with virtme-ng have a `-virtme` suffix to their kernel
version, this allows to easily determine if you're inside a virtme-ng kernel or
if you're using the real host kernel (simply by checking `uname -r`).

It is also possible to generate and inspect a memory dump of the tested kernel
running `virtme-ng -d` from the host, while the test kernel is running.

External kernel modules
=======================

It is possible to recompile and test out-of-tree kernel modules inside the
virtme-ng kernel, simply by building them against the local directory of the
kernel git repository that was used to build and run the kernel.

Default options
===============

Typically, if you always use virtme-ng with an external build server (e.g.,
`virtme-ng --build-host REMOTE_SERVER --build-host-exec-prefix CMD`) you don't
always want to specify these options, so instead, you can simply define them in
~/.virtme-ng.conf under `default_opts` and then simply run `virtme-ng`.

Example (always use an external build server called 'kathleen' and run make
inside a build chroot called 'chroot:lunar-amd64'). To do so, modify the
`default_opts` sections in ~/.config/virtme-ng/virtme-ng.conf as following:

    "default_opts" : {
        "build_host": "kathleen",
        "build_host_exec_prefix": "schroot -c chroot:lunar-amd64 --"
    },

Now you can simply run `virtme-ng` to build your kernel using the external build host,
prepending the exec prefix command when running make.

Troubleshooting
===============

 - If you get permission denied when starting qemu, make sure that your
   username is assigned to the group `kvm` or `libvirt`:
```
  $ groups | grep "kvm\|libvirt"
```

 - When using `--network bridge` to create a bridged network in the guest you
   may get the following error:
```
  ...
  failed to create tun device: Operation not permitted
```

   This is because `qemu-bridge-helper` requires `CAP_NET_ADMIN` permissions.

   To fix this you need to add `allow all` to `/etc/qemu/bridge.conf` and set
   the `CAP_NET_ADMIN` capability to `qemu-bridge-helper`, as following:
```
  $ sudo filecap /usr/lib/qemu/qemu-bridge-helper net_admin
```

 - If the guest fails to start because the host doesn't have enough memory
   available you can specify a different amount of memory using `--memory MB`,
   (this option is passed directly to qemu via -m, default is 4G).

 - If you're testing a kernel for an architecture different than the host, keep
   in mind that you need to use also `--root DIR` to use a specific chroot with
   the binaries compatible with the architecture that you're testing.

   If the chroot doesn't exist in your system virtme-ng will automatically
   create it using the latest daily build Ubuntu cloud image:
```
  $ virtme-ng --arch riscv64 --root ./tmproot
```

 - If the build on a remote build host is failing unexpectedly you may want to
   try cleaning up the remote git repository, running:
```
  $ virtme-ng --clean --build-host HOSTNAME
```

Contributing
============

Please see DCO-1.1.txt.

Credits
=======

virtme-ng is written by Andrea Righi <andrea.righi@canonical.com>

virtme-ng is based on virtme, written by Andy Lutomirski <luto@kernel.org>
([web][korg-web] | [git][korg-git]).

[korg-web]: https://git.kernel.org/cgit/utils/kernel/virtme/virtme.git "virtme on kernel.org"
[korg-git]: git://git.kernel.org/pub/scm/utils/kernel/virtme/virtme.git "git address"
[virtme]: https://github.com/amluto/virtme "virtme"
[virtme-ng-ppa]: https://launchpad.net/~arighi/+archive/ubuntu/virtme-ng "virtme-ng ppa"
