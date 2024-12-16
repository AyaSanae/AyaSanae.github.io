---
title: "Xposed模块本体向Hook类传递信息的研究"
date: '2024-12-16T18:36:36+08:00'
# weight: 1
# aliases: ["/first"]
tags: ["转载","Android","Xposed"]
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
UseHugoToc: false
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
# 引子
在写净眼(一个Xposed模块)时，遇到一个问题，App本体的设置参数如何传给Hook类？
之前我一直用的是文件读写的方式，但在调试中我发现：如果被Hook的应用本身无此权限，那么就无法读到这个文件。
经过多次(两三个小时而已)的查找，找到了一个解决方案。

# XSharedPreferences
XSharedPreferences是XposedBridge的一部分。它可以读取任意应用的SharedPreferences。
於是就可以在App本体中写入SharedPreferences，然后从Hook类中读取…
程序本体中：
```Java
        SharedPreferences mSharedPreferences = getSharedPreferences("data", Context.MODE_WORLD_READABLE);
        data.edit().putString("test","something");
```
Xposed Hook类中：
```Java
        XSharedPreferences xSP=new XSharedPreferences("ml.w568w.test","data");
        xSP.reload();
        xSP.makeWorldReadable();
        String test = xSP.getString("test", "");
```
# 注意
1.必须使用Context.MODE_WORLD_READABLE得到的SharedPreferences，makeWorldReadable()并不能让你可以随意限定SharedPreferences的读取权限。

2.尽可能地不使用PreferenceManager.getDefaultSharedPreferences(this);或者PreferenceActivity得到的SharedPreferences（我在这里被坑了好久…）

3.即便这么做，获得的XSharedPreferences仍然是只读的…因此，Hook类依旧不能向模块本体传递信息…

转自: https://www.w568w.eu.org/a-tiring-xposed-problem.html
