---
title: "Use QEMU's 9pfs to Share Directores on macOS with Linux Guest"
date: 2023-06-27T15:41:43+08:00
draft: false
summary: A guide to share directores on macOS with Linux guest.
tags: ["QEMU", "macOS", "9pfs", "plan 9", "virtfs"]
categories: ["English"]
---

The previous post [Run Ubuntu in QEMU on macOS (Apple Silicon)](/post/qemu-macos-apple-silicon/) instructed us how to
run Ubuntu in QEMU on macOS. Under some scenarios, we want to share directories on macOS with the Ubuntu guest for the
sake of data exchange. This post is a guide to do so.

1. Add the following options to the command `qemu-system-aarch64`.

    ```shell
    -virtfs local,path=/some/path/on/macos,mount_tag=some_mount_tag,security_model=mapped

    ## Example:
    ## Share /tmp/virtfs on macOS with Ubuntu guest and name the mount_tag virtfs.
    # -virtfs local,path=/tmp/virtfs,mount_tag=virtfs,security_model=mapped
    ```
2. After booting the virtual machine, execute the following command to mount the path.

    ```shell
    sudo mount -t 9p -o trans=virtio,msize=524288 some_mount_tag /some/mount/point/on/linux

    ## Example:
    ## On Ubuntu
    # mkdir /tmp/virtfs
    # sudo mount -t 9p -o trans=virtio,msize=524288 virtfs /tmp/virtfs
    ```

3. If the step `2` is executed successfully, we can use the command `mount` to check whether the 9pfs is mounted
   successfully.

    ```shell
    # Example:
    # We should see the virtfs is mounted on /tmp/virtfs with type 9p.
    $ mount

    sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
    proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
    udev on /dev type devtmpfs (rw,nosuid,relatime,size=1928632k,nr_inodes=482158,mode=755,inode64)
    devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
    ...
    virtfs on /tmp/virtfs type 9p (rw,relatime,sync,dirsync,access=client,msize=512000,trans=virtio)
    ```

4. **(Optional)** We can modify `/etc/fstab` to persist the mount point by adding the following line to `/etc/fstab`.

    ```shell
    some_mount_tag /some/mount/point/on/linux 9p _netdev,trans=virtio,msize=524288 0 0

    ## Example:
    # virtfs /tmp/virtfs 9p _netdev,trans=virtio,msize=524288 0 0
    ```

# References

1. [Documentation/9psetup](https://wiki.qemu.org/Documentation/9psetup)
