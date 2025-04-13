---
title: "Yocto custom UBI image guide in detail"
date: 2025-04-13T22:06:36+02:00
categories:
  - blog
tags:
  - Embedded Linux
  - Yocto
  - NAND
  - UBI
  - squashfs
  - NXP
  - i.MX
---

In this blog, I will discuss something new I found based on [my previous blog],
as well as another [Marcus Folkesson's blog].

I **strongly recommend** you to go through [Marcus Folkesson's blog] first,
so that you get a basic idea how to integrate multiple volumes in Yocto for a UBI device.

I utilized his work and started to integrate for a UBI device with two volumes of
squashfs and one volume of overlay in UBIFS.

As I mentioned in [my previous blog],
the reason of picking squashfs is for space efficiency.
Two symmetric dual rootfs (squashfs) and an overlay (ubifs) fit into the limited NAND.
There are good explanations in [squashfs-over-ubi] and [RO block devices on top of UBI volumes].

I started working on creating custom `mybui.bblass` according to [Marcus Folkesson's blog].
However, I found out one thing is not really working in Yocto:

> We need to create our own class in order to override that function.

Yes, we need to "override" `write_ubi_config()` which is defined in Yocto `image_types.bbclass`.
However, since `write_ubi_config()` comes from `image_types.bbclass`, it is not override-able.

After some debugging, I realized that I need to create a complete `mybui.bblass`
with different naming of functions, which enables a complete build.

Here is a complete example of `mybui.bblass`:

```bash
IMAGE_FSTYPES = "squashfs myubi"
UBI_IMGTYPE ?= "squashfs"

do_image_myubi[depends] += "mtd-utils-native:do_populate_sysroot"

write_myubi_config() {
    cat <<EOF > ubinize-${IMAGE_NAME}.cfg
[rootfs0]
mode=ubi
image=${IMGDEPLOYDIR}/${IMAGE_NAME}.squashfs
vol_id=0
vol_size=56MiB
vol_type=static
vol_name=rootfs0

[rootfs1]
mode=ubi
image=${IMGDEPLOYDIR}/${IMAGE_NAME}.squashfs
vol_id=1
vol_size=56MiB
vol_type=static
vol_name=rootfs1

[overlayfs]
mode=ubi
vol_id=2
vol_size=400MiB
vol_type=dynamic
vol_name=overlayfs
vol_flags=autoresize

EOF
}

IMAGE_TYPEDEP:myubi = "squashfs"
IMAGE_CMD:myubi () {
    # Added prompt error message for ubi image creation.
    if [ -z "${UBINIZE_ARGS}" ]; then
        bbfatal "UBINIZE_ARGS has to be set, see http://www.linux-mtd.infradead.org/faq/ubifs.html for details"
    fi

    write_myubi_config

    ubinize -o ${IMGDEPLOYDIR}/${IMAGE_NAME}.myubi ${UBINIZE_ARGS} ubinize-${IMAGE_NAME}.cfg

    # Cleanup cfg file
    mv ubinize-${IMAGE_NAME}.cfg ${IMGDEPLOYDIR}/
}
```

And in your actual `my-image.bb` (suppose your `IMAGE_NAME` is `my-image`), you need to have:

```bash
IMAGE_CLASSES = "myubi"
inherit core-image myubi
```

Here I am using `core-image` since I only need a bare minimal `core-image-minimal`.
You can choose other image inheritance instead of `core-image` based on your need.

After `bitbake my-image`, you will find your final rootfs image in `${IMGDEPLOYDIR}` named as `${IMAGE_NAME}.myubi`.

Since I am working on NXP i.MX device, I can flash the whole UBI image with little [uuu ubi customization]. And it works much smoother than [`uuu`'s official example] with tar extraction.

Hope this little guide will help you when you encounter problems with UBI volume customization in Yocto.

## **Caveat**

In my example `mybui.bblass`, I don't have `ubifs` in `IMAGE_FSTYPES` or `UBI_IMGTYPE`,
this is because my overlay is created dynamically in `ubinize` with `vol_type=dynamic` and empty `image` config.

But if you have your own volumes with images which are in UBIFS, then you need to have
`ubifs` in `IMAGE_FSTYPES`, and correctly define your `MKUBIFS_ARGS`, and `UBINIZE_ARGS`.
And [Marcus Folkesson's blog] has good examples for it.

Another thing is that Yocto default environment variables provides:

```bash
UBI_VOLNAME ?= "${MACHINE}-rootfs"
UBI_VOLTYPE ?= "dynamic"
UBI_IMGTYPE ?= "ubifs"
```

But these variables are not particularly useful in our cases since every UBI volume
can have its own volume name, volume type, and image type.
So one set of environment variables cannot not rule out all our needs.
It's just good enough to define your own `write_myubi_config()` in detail.

[squashfs-over-ubi]: http://linux-mtd.infradead.org/faq/ubi.html#L_squashfs_over_ubi
[RO block devices on top of UBI volumes]: http://linux-mtd.infradead.org/doc/ubi.html#L_ubiblock
[my previous blog]: https://swechencheng.github.io/blog/custom-upperdir-in-meta-readonly-rootfs-overlay/
[Marcus Folkesson's blog]: https://www.marcusfolkesson.se/blog/ubi-volumes-in-yocto/
[uuu ubi customization]: https://github.com/nxp-imx/mfgtools/issues/433#issuecomment-2663403128
[`uuu`'s official example]: https://github.com/nxp-imx/mfgtools/wiki/Sample-scripts#burn-nand-flash-by-linux-kernel
