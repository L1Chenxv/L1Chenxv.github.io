---

layout: fragment
title: github pages 部署失败
tags: [github]
description: 
keywords: github

---

### bug

> Deploy to GitHub pages failing

今天写一篇新博客快快乐乐的上传到github，自己登陆网站直接提示【404】，去项目action看了下所有自动部署的 git 操作功能 pages-build-deployment 都没有启动。

自己又找了一个月之前的部署任务手动部署了下，也是部署失败。这个任务在当时是构建、部署没有异常，线上也运行良好。

![1746958833283](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/随记/1746958833283.2obqtuzu9v.webp)



### 解决

后面谷歌了下有没有解决方案，发现有老哥遇到了相同问题，他是因为学生账户权限到期引发部署失败，最后通过把可见权限从private改为了public解决。

想来我也是最近学生包的权限过期了，就把项目可见权限从private改为了public，尝试了下确实有用！

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/随记/image.86tva0f4ff.webp)

### 总结

github的坑点还是比较多的...



