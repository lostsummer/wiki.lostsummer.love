---
title: "SVN分支操作"
date: 2017-04-25 17:10
---

以下操作都是在本地工作目录 C 中


* __工作目录从主干转到分支（当前工作目录在主干）__




     svn switch http://10.10.10.57:6688/svn/myaudit_c/brunches/muti-pretreat/src/cplusplus/c




* __工作目录从分支转到主干（当前工作目录在分支）__




    svn switch http://10.10.10.57:6688/svn/myaudit_c/src/cplusplus/c




* __将主干代码的修改合并到分支（当前工作目录在分支）__




     svn update
     svn merge -r5746:HEAD http://10.10.10.57:6688/svn/myaudit_c/src/cplusplus/c


解决冲突之后


    svn commit -m "comment"


* __将分支的修改合并到主干（当前工作目录在主干）__




     svn update
     svn merge -r5746:HEAD http://10.10.10.57:6688/svn/myaudit_c/brunches/muti-pretreat/src/cplusplus/c


   解决冲突之后





     svn commit -m "comment"


* __一个有用的命令，丢弃工作目录内所有修改（还原到 svn 库中当前版本）__




    svn revert --depth=infinity .


<br>


__当前版本情况__

![](http://img.lostsummer.love/wiki-img/svn-branch.png)
