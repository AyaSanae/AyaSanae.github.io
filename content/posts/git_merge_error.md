---
title: "解决Git合并报-refusing to merge unrelated histories"
date: '2020-04-07T16:25:17+08:00'
# weight: 1
# aliases: ["/first"]
tags: ["Git","Error"]
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
## 原因
导致这种错误是因为两个仓库没有关系,所以在合并的时候远端仓库觉得和本地仓库没有关系,就报了refusing to merge unrelated histories

## 解决方法
加上--allow-unrelated-histories选项,将两个不相干仓库强行合并

```
git pull origin master --allow-unrelated-histories
```
