---
title: "Custom `upperdir` in meta-readonly-rootfs-overlay"
date: 2025-03-24T16:16:33+01:00
categories:
  - blog
tags:
  - Embedded Linux
  - Yocto
  - meta-readonly-rootfs-overlay
---

I have been working and designing a new Embedded Linux with a brand new upgrade mechanism on an 7-10 year-old legacy Embedded Linux devices which had no atomic `rootfs` upgrade mechanism.
the legacy system has a proprietary package manager which is working fine and I have to keep the compatibility with it.
It is a complicated task, however the modern Yocto gives almost everything I need:
- [Stefano Babic's meta-swupdate]
- [Marcus Folkesson's meta-readonly-rootfs-overlay]

`meta-swupdate` gives the whole infrastructure of doing atomic `rootfs` upgrades in the future.
`meta-readonly-rootfs-overlay` gives the possibility of keeping backward compatibility with legacy system... Sounds too easy is not it?
Oh well, this kind of hybrid design is challenging and complex to manage, but that is not the scope of this blog for now.

Let me come back to the current need:
E.g., if the user has A/B partition scheme: A & B RO rootfs,
but only have one disk/volume for the rootrw due to limited partitioning space,
and the user wants to switch between A/B RO `rootfs` meanwhile keep the overlayfs matching the RO `rootfs` that is being activated,
then the user can define their custom `upperdir` name via kernel `cmdline`.

I have made [a PR] to make it possible for user to define their own `upperdir` location in rootrw.
This gives the user more control over the `upperdir`(s) and the possibility to manage multiple versions of `upperdir`.

The usage is, e.g. in my particular kernel `cmdline`, if the system boots from rootfs A:
```bash
console=ttymxc0,115200 ubi.mtd=nandubi ubi.block=0,rootfs0 root=/dev/ubiblock0_0 rootfstype=squashfs rootoptions=ro rootrw=ubi0:overlayfs rootrwfstype=ubifs rootrwoptions=rw rootrwreset=no rootrwupperdir=upperdir0 init=/init mtdparts=gpmi-nand:4m(nandboot),-(nandubi)
```

If the system boots from rootfs B:
```bash
console=ttymxc0,115200 ubi.mtd=nandubi ubi.block=0,rootfs1 root=/dev/ubiblock0_1 rootfstype=squashfs rootoptions=ro rootrw=ubi0:overlayfs rootrwfstype=ubifs rootrwoptions=rw rootrwreset=no rootrwupperdir=upperdir1 init=/init mtdparts=gpmi-nand:4m(nandboot),-(nandubi)
```

Just to clarify a bit:
This device have a small raw NAND where the system runs, thus we use underlying MTD partitions and UBI device.
The RO rootfs is running squashfs over UBI block device emulation for saving spaces. And an RW "overlay" sits on top of that.

You can read more how basically it works in [Marcus Folkesson's blog].

Last but not least, people may ask:
- How to cleanup the old `upperdir`s?
  - First choice is to use `rootrwreset` in [meta-readonly-rootfs-overlay Details] to clear all `upperdir`s.
  - But if you have custom requirements such as to retain only certain version of `upperdir`s, then you have to implement your own logic.

[Stefano Babic's meta-swupdate]: https://github.com/sbabic/meta-swupdate
[Marcus Folkesson's meta-readonly-rootfs-overlay]: https://github.com/marcusfolkesson/meta-readonly-rootfs-overlay
[Marcus Folkesson's blog]: https://www.marcusfolkesson.se/blog/meta-readonly-rootfs-overlay/
[meta-readonly-rootfs-overlay Details]: https://github.com/marcusfolkesson/meta-readonly-rootfs-overlay/tree/master?tab=readme-ov-file#details
[a PR]: https://github.com/marcusfolkesson/meta-readonly-rootfs-overlay/pull/8
