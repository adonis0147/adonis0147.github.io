---
title: "Set QEMU Networking up on macOS by socket_vmnet"
date: 2024-05-26T23:14:10+08:00
draft: false
summary: A guide to set QEMU networking up on macOS by socket_vmnet.
tags: ["QEMU", "macOS", "socket_vmnet"]
categories: ["English"]
---

The previous post [Run Ubuntu in QEMU on macOS (Apple Silicon)](/post/qemu-macos-apple-silicon/) instructed us how to
set QEMU networking up by [vde_vmnet](https://github.com/lima-vm/vde_vmnet). However, the 
[vde_vmnet](https://github.com/lima-vm/vde_vmnet) was deprecated. This post is a guide to set QEMU networking up by the
successor [socket_vmnet](https://github.com/lima-vm/socket_vmnet).

# Prerequisite

1. ubuntu-24.04-live-server-arm64.iso
2. QEMU >= 9.0
3. [socket_vmnet](https://github.com/lima-vm/socket_vmnet) >= 1.1.4
4. macOS >= Sonoma (Apple Silicon)

## Preparation

```shell
# Download the Ubuntu live cd from the official website.
# https://ubuntu.com/download/server/arm

# Install QEMU by Homebrew.
brew install qemu

# Install socket_vmnet.
brew install socket_vmnet

# Install the launchd service
brew tap homebrew/services
# sudo is necessary for the next line
sudo brew services start socket_vmnet
```

## Test

```shell
ping 192.168.105.1
```

If the test is failed, try to restart the service by the following command.

```shell
sudo brew services restart socket_vmnet
```

# Install Ubuntu

1. Download QEMU EFI.

    ```shell
    curl -L https://releases.linaro.org/components/kernel/uefi-linaro/latest/release/qemu64/QEMU_EFI.fd -o QEMU_EFI.fd

    ```

2. Create an image.

    ```shell
    qemu-img create -f raw -o preallocation=full ubuntu.img 40G
    ```

3. Boot QEMU to install Ubuntu.

    ```shell
    socket_vmnet_client /opt/homebrew/var/run/socket_vmnet \
        qemu-system-aarch64 \
            -machine virt,accel=hvf \
            -cpu host \
            -smp 8 \
            -m 8G \
            -device virtio-net-pci,netdev=net0 -netdev socket,id=net0,fd=3 \
            -drive if=virtio,format=raw,file=./ubuntu.img \
            -cdrom <path/to/ubuntu iso> \
            -nographic \
            -bios QEMU_EFI.fd
    ```

# Boot Ubuntu in QEMU

```shell
socket_vmnet_client /opt/homebrew/var/run/socket_vmnet \
    qemu-system-aarch64 \
        -machine virt,accel=hvf \
        -cpu host \
        -smp 8 \
        -m 8G \
        -device virtio-net-pci,netdev=net0 -netdev socket,id=net0,fd=3 \
        -drive if=virtio,format=raw,file=./ubuntu.img \
        -nographic \
        -bios QEMU_EFI.fd
```

# Log into the server by SSH

```shell
ssh 192.168.105.2
```
