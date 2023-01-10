---
title: "Run Ubuntu in QEMU on MacOS (Apple Silicon)"
date: 2022-05-28T11:48:49+08:00
draft: false
summary: A guide to run Ubuntu in QEMU on MacOS (Apple Silicon).
tags: ["QEMU", "MacOS", "Apple Silicon", "VDE", "VDE_VMNET", "M1"]
categories: ["English"]
---

This post is a guide to run Ubuntu in QEMU on MacOS (Apple Silicon).

The laptop in this post is `MacBook Pro (13-inch, M1, 2020)`.

The OS is `MacOS 12.4 21F79 arm64`.

# Prerequisite

1. ubuntu-22.04-live-server-arm64.iso
2. QEMU >= 7.0
3. [vde_vmnet](https://github.com/lima-vm/vde_vmnet) >= v0.6.0
4. MacOS >= Monterey (Apple Silicon)

## Preparation

```shell
# Download the Ubuntu live cd from the official website.
# https://ubuntu.com/download/server/arm

# Install QEMU by Homebrew.
brew install qemu

# Install vde_vmnet.
git clone https://github.com/lima-vm/vde_vmnet
cd vde_vmnet
git config --global --add safe.directory "$(pwd)"
sudo make PREFIX=/opt/vde install
```

## Test

```shell
ping 192.168.105.1
```

If the test is failed, try to reload the launch plists manually to make sure the processes (`vde_switch` and `vde_vmnet`) are running.

```shell
# Unload
sudo launchctl unload -w "/Library/LaunchDaemons/io.github.lima-vm.vde_vmnet.plist"
sudo launchctl unload -w "/Library/LaunchDaemons/io.github.virtualsquare.vde-2.vde_switch.plist"

# Load (order matters)
sudo launchctl load -w "/Library/LaunchDaemons/io.github.virtualsquare.vde-2.vde_switch.plist"
sudo launchctl load -w "/Library/LaunchDaemons/io.github.lima-vm.vde_vmnet.plist"
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
    qemu-system-aarch64 \
            -machine virt,accel=hvf \
            -cpu host \
            -smp 8 \
            -m 8G \
            -drive if=virtio,format=raw,file=./ubuntu.img \
            -cdrom <path/to/ubuntu iso> \
            -nographic \
            -bios QEMU_EFI.fd
    ```

# Boot Ubuntu in QEMU

```shell
qemu-system-aarch64 \
        -machine virt,accel=hvf \
        -cpu host \
        -smp 8 \
        -m 8G \
        -nic vde,model=virtio,sock=/var/run/vde.ctl \
        -drive if=virtio,format=raw,file=./ubuntu.img \
        -nographic \
        -bios QEMU_EFI.fd
```

## Test

Use the command `lspci` to check whether the network device is ready.

```shell
$ lspci

00:00.0 Host bridge: Red Hat, Inc. QEMU PCIe Host bridge
00:01.0 Ethernet controller: Red Hat, Inc. Virtio network device
00:02.0 SCSI storage controller: Red Hat, Inc. Virtio block device
```

# Configure the networking

1. Get the name of the network device.

    ```shell
    $ ip link

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
        link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    ```

    We can see the name of the network device is `enp0s1`.

2. Configure the netplan.

    i. Create the configuration file.
    ```shell
    sudo touch /etc/netplan/enp0s1-config.yaml
    ```

    ii. Edit the `/etc/netplan/enp0s1-config.yaml` using the following content.
    ```yaml
    network:
      version: 2
      ethernets:
        enp0s1:
          dhcp4: false
          addresses: [192.168.105.2/24]
          nameservers:
            addresses: [192.168.105.1]
          routes:
            - to: default
              via: 192.168.105.1
    ```

3. Test the netplan.

    ```shell
    sudo netplan try

    # Ping the gateway.
    ping 192.168.105.1
    ```

# Log into the server by SSH

```shell
ssh 192.168.105.2
```
