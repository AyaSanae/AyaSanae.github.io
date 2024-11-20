---
title: "在Ubuntu上创建SWAP"
date: '2021-07-23T16:20:34+08:00'
# weight: 1
# aliases: ["/first"]
tags: ["Linux"]
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

## 0x01 创建swap文件
```
sudo dd if=/dev/zero of=/Swapfile bs=1M count=2048

//if=input file，这里表示往文件中写0
//of=output file，这是在根目录下创建Swapfile文件
//bs=bytes，设置每个块大小，后面的数值大,写的速度相对快一点,但也不是无限的.
//count表示有多少个这种块
//新建的Swap文件大小为bs*count的大小
```

## 0x02 格式化并启用swap文件
```
sudo mkswap /Swapfile    //格式化根目录下的Swapfile文件
sudo swapon /Swapfile    //启用根目录下的Swapfile文件
```

## 0x03 查看结果
```
free -m //当Swap不为0时表示成功
```

Enjoy.
