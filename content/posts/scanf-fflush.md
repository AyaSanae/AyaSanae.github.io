---
title: "scanf()与循环与缓冲区"
date: '2022-10-27T16:33:29+08:00'
# weight: 1
# aliases: ["/first"]
tags: ["C","ProgramLanguage"]
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


## Source
```c
    float course_1=-1,course_2=-1,result;
    while(1){
        printf("Please enter the score of the first course:");
        scanf("%f",&course_1);
            if(course_1 < 0 || course_1 > 100){
                    printf("Please enter the correct score!\n");
                    continue;
            }
        printf("\nPlease enter the score of the second course:");
        scanf("%f",&course_2);
            if(course_2 < 0 || course_2 > 100){
                    printf("Please enter the correct score!\n");
                    continue;
            }
            break;
    }       
    result = (course_1 + course_2)/2;
    printf("\nThe average score is:%.1f",result);
}    
```
## 输出

如果输入的值为非字符时结果为:
```
Please enter the score of the first course:20

Please enter the score of the second course:30

The average score is:25.0
```
如果输入的值为字符时结果为:
```
Please enter the score of the first course:a

Please enter the score of the first course:Please enter the correct score!
Please enter the score of the first course:Please enter the correct score!
Please enter the score of the first course:Please enter the correct score!
...
```

## 原因

因为scanf里占位符是%d,也就是录入缓冲区中的整形数字.
当缓冲区有字符,scanf便会读取,但是不是整形就不会录入.
        
第一次输入了a,是非整形,scanf略过了但是又没有清空缓冲区,第二次便不会要求我输入
因为缓冲区里有字符,又略过,如此便是死循环

## 解决
可以使用fflush(stdin)来清空缓冲区.
```c
   float course_1=-1,course_2=-1,result;
    while(1){
        printf("Please enter the score of the first course:");
        scanf("%f",&course_1);
        fflush(stdin)
            if(course_1 < 0 || course_1 > 100){
                    printf("Please enter the correct score!\n");
                    continue;
            }
        printf("\nPlease enter the score of the second course:");
        scanf("%f",&course_2);
        fflush(stdin)
            if(course_2 < 0 || course_2 > 100){
                    printf("Please enter the correct score!\n");
                    continue;
            }
            break;
    }       
    result = (course_1 + course_2)/2;
    printf("\nThe average score is:%.1f",result);
}    
```

这时候程序就正常了.
