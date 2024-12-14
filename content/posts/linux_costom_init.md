---
title: "在内核启动后使用自己的init"
date: '2024-12-14T15:14:50+08:00'
# weight: 1
# aliases: ["/first"]
tags: ["Linux","Kernel"]
summary: " "
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
canonicalURL: "https://canonical.url/to/page"
disableShare: true
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

从boot里偷一份内核,放到qemu里运行,把init换成自己的.
启动到initramfs后,尝试挂载sda,发现挂不上,看了眼dmesg发现Can't open blockdev.
因为我虚拟的sda设备是Ext4文件系统,同时qemu用了Virtual IO.
而我从boot偷过来的kernel,是没Virtual IO和Ext4驱动的,他们以内核模块的方式被编译,我认为可以把当前系统里面的相关的内核模块一并拿到initramfs加载了,但是开考虑到相关Kernel Module还是有好几个依赖的,就没做(懒).

于是乎,我拉了个Linux内核并用LLVM直接编译了相关驱动进内核,配置直接从当前运行系统中的/proc/config.gz拿.

需要改的有:
```
CONFIG_BLK_DEV=y
CONFIG_SCSI=y
CONFIG_SCSI_LOWLEVEL=y
CONFIG_SCSI_VIRTIO=y
CONFIG_VIRTIO_BLK=y
CONFIG_ATA=y
CONFIG_ATA_PIIX=y
```

顺带附上我的Makefile和INIT脚本:
### Makefile
```make
# Requires statically linked busybox in $PATH

INIT := /init
IMG := build/disk.img
MOUNT := $(shell mktemp -d)
K := $(shell uname -r)

all: initramfs fsroot

initramfs:
# Copy kernel and busybox from the host system
	@mkdir -p build/initramfs/bin
	# sudo bash -c "cp /boot/vmlinuz-linux build/ && chmod 666 build/vmlinuz-linux"
	cp bzImage build/vmlinuz-linux && chmod 666 build/vmlinuz-linux
	chmod +x init && cp init build/initramfs/
	cp $(shell which busybox) build/initramfs/bin/

# Pack build/initramfs as gzipped cpio archive
	cd build/initramfs && \
	  find . -print0 \
	  | cpio --null -ov --format=newc \
	  | gzip -9 > ../initramfs.cpio.gz

fsroot:
	mkdir -p fsroot/modules
	cp e1000.ko fsroot/modules/
#
	dd if=/dev/zero of=$(IMG) bs=1M count=64
	mkfs.ext4 -F $(IMG)
	sudo mount $(IMG) $(MOUNT)
	cd fsroot && chmod +x ./init && sudo cp -r * $(MOUNT)
	sudo umount $(MOUNT)

run:
# Run QEMU with the installed kernel and generated initramfs
	qemu-system-x86_64 \
	  -serial mon:stdio -vga std \
	  -drive file=$(IMG),format=raw,index=0,media=disk \
	  -netdev user,id=net0,hostfwd=tcp:127.0.0.1:8080-:8080 -device e1000,netdev=net0 \
	  -kernel build/vmlinuz-linux \
	  -initrd build/initramfs.cpio.gz \
	  -machine accel=kvm:tcg \
	  -append "console=ttyS0 quiet rdinit=$(INIT)"

clean:
	rm -rf build fsroot/modules

.PHONY: initramfs run clean fsroot
```
### initramfs运行的INIT:
```SH
#!/bin/busybox sh

# At this point, we only have:
#   /bin/busybox - the binary
#   /dev/console - the console device
BB=/bin/busybox

# Delete this file
# (Yes, we can do this on UNIX! The file is "removed"
# after the last reference, the fd, is gone.)
$BB rm /init

$BB find / -type f
$BB echo "Unlimited power!!"
# $BB poweroff -f
# ----------------------------------------------------
 
# "Create" command-line tools by making symbolic links
#   try: busybox --list
for cmd in $($BB --list); do
    $BB ln -s $BB /bin/$cmd
done

# Mount procfs and sysfs
mkdir -p /proc && mount -t proc  none /proc
mkdir -p /sys  && mount -t sysfs none /sys

# Create devices
mknod /dev/random  c 1 8
mknod /dev/urandom c 1 9
mknod /dev/null    c 1 3
mknod /dev/tty     c 4 1
mknod /dev/sda     b 8 0

echo -e "\033[31mInit OK; launch a shell (initramfs).\033[0m"

busybox sh

# Display a countdown

echo -e "\n\n"
echo -e "\033[31mSwitch root in...\033[0m"
for sec in $(seq 3 -1 1); do
    echo $sec; sleep 1
done

# Switch root to /newroot (a real file system)
N=/newroot

mkdir -p $N
mount -t ext4 /dev/sda $N

mkdir -p $N/bin
cp $BB $N/bin/
cp $BB /init
exec switch_root /newroot/ /init
```
### switch_root 后运行的INIT:
```SH
#!/bin/busybox sh

# Now, "/" is the virtual disk--for real
# systems, we expect installed binaries.
BB=/bin/busybox
for cmd in $($BB --list); do
    $BB ln -s $BB /bin/$cmd
done

mkdir -p /proc && mount -t proc  none /proc
mkdir -p /sys  && mount -t sysfs none /sys
mkdir -p /tmp  && mount -t tmpfs none /tmp

mkdir -p /dev
mknod /dev/random  c 1 8
mknod /dev/urandom c 1 9
mknod /dev/null    c 1 3
mknod /dev/tty     c 4 1
mknod /dev/sda     b 8 0

# Configure network
insmod /modules/e1000.ko
ip link set lo up
ip link set eth0 up
ip addr add 10.0.2.15 dev eth0
ip route add 10.0.2.0/24 dev eth0

echo -e "Hello AyaSanae"
# Prepare the terminal
# echo -e "\033[31mGoodbye, QEMU Console!\033[0m"
# echo -e "\033[H\033[2J" >/dev/tty
# echo -e "JYY's minimal Linux" >/dev/tty
sh
# Switch to terminal
# setsid /bin/sh </dev/tty >/dev/tty 2>&1
# 
# sync
# poweroff -f
```
