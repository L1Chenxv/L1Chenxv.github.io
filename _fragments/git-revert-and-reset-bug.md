---
layout: fragment
title: git pages 搭建失败
tags: [git]
description: 搭建失败，提示有一个部署进程正在执行
keywords: Java, IDEA
---

回滚操作导致有个branch一直在部署，但是找不到。

尝试各种方法都不行：

 - 重启idea
 - 重启电脑
 - 把deploy完成的分支，一条一条 `accept` 到当前分支

最后想到实习时候进行回滚操作之后，频繁提示代码冲突

猜想可能是revert或者reset操作造成

最后重新构建`repository`才解决。。。

Done~！