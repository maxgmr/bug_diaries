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

Unfortunately, this worked perfectly! This likely means the bug boils down to a race condition while `apt` tries to upgrade the two packages.

## The Potential Solution

`rust-X.Y-src.links.in` links its installed files to the versioned `rustlib` path:

```
usr/src/rustc-${env:RUST_LONG_VERSION} usr/lib/rust-${env:RUST_VERSION}/rustlib/src/rust
usr/src/rustc-${env:RUST_LONG_VERSION} usr/lib/rust-${env:RUST_VERSION}/lib/rustlib/src/rust
```

This means that `rustc.links` already achieves the desired end result below, making `rust-src.links` superfluous:

```
/usr/lib/rust-${env:RUST_VERSION}/lib/rustlib	/usr/lib/rustlib
```

## Another Error!

It turns out that `rust-src.links` is broken in more ways than one! `rust-src.links` creates a symlink to `rustc-RUST_VERSION`...

```
/usr/src/rustc-${env:RUST_VERSION} /usr/lib/rustlib/src/rust
```

...but `rust-X.Y-src.install.in` installs files to `rustc-RUST_LONG_VERSION`!

```
src             usr/src/rustc-${env:RUST_LONG_VERSION}
...etc...
```

This means that the symlink which gets installed is actually broken _regardless_ of any package ownership conflicts.

## Killing Two Birds With One Stone

Both problems should be solvable by simply emptying `debian/rust-src.links`:

```diff
--- a/debian/rust-src.links
+++ b/debian/rust-src.links
@@ -1 +0,0 @@
-/usr/src/rustc-${env:RUST_VERSION} /usr/lib/rustlib/src/rust
```

This removes both the conflict _and_ the bad symlink.

## Testing the solution

Let's double-check to make sure we haven't somehow missed out on any files (although this is unlikely, considering the only thing we removed was already broken)!

Control group (in an LXC container):

```shell
apt update && apt upgrade -y
apt install rustc rust-src tree
```

Then, check the output of the following command:

```shell
tree -l /usr/lib/rustlib
```

Experimental group (also LXC):

```shell
apt update && apt upgrade -y
add-apt-repository ppa:maxgmr/rust-defaults-lp2119014
apt update
apt install rustc=1.85.1ubuntu2\~ppa1 rust-src=1.85.1ubuntu2\~ppa1 tree
```

Compare with the previous output:

```shell
tree -l /usr/lib/rustlib
```

Now the symlink actually works!

## Caveat

Note that now, if one installs _just_ `rust-src`, there will be no `/usr/lib/rustlib`. This shouldn't break anyone's systems, though, considering the fact that this symlink was broken in the first place...
