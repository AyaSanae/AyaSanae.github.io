---
title: "解决Windows10更新或重装后kali引导丢失"
date: '2020-05-04T16:29:22+08:00'
# weight: 1
# aliases: ["/first"]
tags: ["Linux","Error"]
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

## 正文:
本文使用Grub来修复引导
你需要准备一个Kali Live,这里我用我的u盘,将kali Live安装镜像写到启动盘里
过程就不演示了

开机BIOS引导启动盘,进入Kali Live
打开终端
`sudo fdisk -l`
查找装有Kali的硬盘
以下是我的报告
```
Disk /dev/sdc: 111.81 GiB, 120034123776 bytes, 234441648 sectors
Disk model: RUIREN TECH SSD 
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x3e308e49

Device     Boot     Start       End   Sectors   Size Id Type
/dev/sdc1            2048 221976575 221974528 105.9G 83 Linux
/dev/sdc2       221978622 234440703  12462082     6G  5 Extended
/dev/sdc5       221978624 234440703  12462080     6G 82 Linux swap / Solaris


Disk /dev/sda: 465.78 GiB, 500107862016 bytes, 976773168 sectors
Disk model: TOSHIBA MK5061GS
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xf8aef8ae

Device     Boot     Start       End   Sectors   Size Id Type
/dev/sda1              63 419441084 419441022   200G  7 HPFS/NTFS/exFAT
/dev/sda2       419441085 976768940 557327856 265.8G  7 HPFS/NTFS/exFAT


Disk /dev/sdb: 111.81 GiB, 120034123776 bytes, 234441648 sectors
Disk model: INTEL SSDSC2BW12
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xfb4efb4e

Device     Boot Start       End   Sectors   Size Id Type
/dev/sdb1  *       63 234437070 234437008 111.8G  7 HPFS/NTFS/exFAT
```
我的kali装在了/dev/sdc硬盘里的第一分区也就是/dev/sdc1(tip:可根据分区类型和大小来判断)
### 挂载必要的目录
```
sudo mount /dev/sdc1 /mnt/           (这里的/dev/sdc1换成你Kali所在的分区)
sudo mount —bind /sys /mnt/sys
sudo mount —bind /proc /mnt/proc
sudo mount –bind /dev /mnt/dev
```

### 使用chroot引导Kali
`sudo chroot /mnt`
这时候会进入已经安装在硬盘的Kali

### 安装Grub引导
```
sudo grub-install /dev/sda    (tip:这里的/dev/sda是Grub的安装路径我将它装到带有windows的硬盘上)
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### 收工
`exit`
`sudo reboot`
拔出u盘,静等Grub的引导出现~
