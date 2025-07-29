## The Problem

As described in the [bug report](https://pad.lv/2119014), when trying to upgrade `rust-defaults` from `1.84.0` to `1.85.1`, the upgrade fails with the following message:

```
Unpacking rustc (1.85.1ubuntu1) over (1.84.0ubuntu1) ...
dpkg: error processing archive /tmp/apt-dpkg-install-Bacnwr/155-rustc_1.85.1ubuntu1_riscv64.deb (--unpack):
 trying to overwrite '/usr/lib/rustlib', which is also in package rust-src (1.84.0ubuntu1)
```

## Tracking Down the Issue

This is not a typical file conflict issue—file conflicts don't really happen on directories. We need to figure out _what_ the packages are actually trying to do.

### Viewing the installed files

First, let's look at what files are actually being installed by the two binary packages:

```shell
dpkg-deb -c rust-src_1.85.1ubuntu1_all.deb
dpkg-deb -c rustc_1.85.1ubuntu1_amd64.deb
```

One potential culprit can be spotted immediately.

`rustc_1.85.1`:

```
lrwxrwxrwx root/root         0 2025-06-20 12:22 ./usr/lib/rustlib -> rust-1.85/lib/rustlib
```

`rust-src_1.85.1`:

```
drwxr-xr-x root/root         0 2025-06-20 12:22 ./usr/lib/rustlib/
drwxr-xr-x root/root         0 2025-06-20 12:22 ./usr/lib/rustlib/src/
lrwxrwxrwx root/root         0 2025-06-20 12:22 ./usr/lib/rustlib/src/rust -> ../../../src/rustc-1.85
```

I'm still relatively new to this stuff, but instinctively it doesn't seem right that `rust-src_1.81.5` is trying to install files to a symlink which may or may not exist yet.

## Attempting to Reproduce

Let's try to reproduce the bug locally, so we can try things out and see if they work.

```shell
lxc launch ubuntu-daily:25.10 questing
lxc shell questing
```

Within the LXC container, let's manually install the old `rustc` and `rust-src` versions:

```shell
wget http://launchpadlibrarian.net/777329063/rust-1.84-src_1.84.1+dfsg0ubuntu1-0ubuntu1_all.deb
wget http://launchpadlibrarian.net/777329070/rustc-1.84_1.84.1+dfsg0ubuntu1-0ubuntu1_amd64.deb
wget http://launchpadlibrarian.net/771878306/rust-src_1.84.0ubuntu1_all.deb
wget http://launchpadlibrarian.net/771878307/rustc_1.84.0ubuntu1_amd64.deb

```

```shell
apt install binutils libllvm19 libstd-rust-1.84-dev gcc libc-dev
dpkg -i rust-1.84-src_1.84.1+dfsg0ubuntu1-0ubuntu1_all.deb
dpkg -i rust-src_1.84.0ubuntu1_all.deb
dpkg -i rustc-1.84_1.84.1+dfsg0ubuntu1-0ubuntu1_amd64.deb
dpkg -i rustc_1.84.0ubuntu1_amd64.deb
```

Now, let's try to update:

```shell
apt update && apt upgrade
```

Unfortunately, this worked perfectly!
