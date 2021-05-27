---
title: 移除git中的大文件
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - git
tags:
 - 效率提升
publish: true
---


github上有官方的参考，地址：[Removing sensitive data from a repository](https://help.github.com/en/articles/removing-sensitive-data-from-a-repository)

实战也有，地址：[Git从库中移除已删除大文件 - 白杨的专栏 - 博客频道 - CSDN.NET](https://blog.csdn.net/zcf1002797280/article/details/50723783)

也可以参考：[从 Git 仓库中永久删除文件或目录](https://www.tuicool.com/articles/uq2yMvV)

还有这篇 https://harttle.land/2016/03/22/purge-large-files-in-gitrepo.html

以及这篇 https://www.cnblogs.com/congyucn/p/9309734.html

经过我实际操作，如果文件名有空格，则需要用双引号引起来，主要命令如下，按照顺序操作即可：
```bash
git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch "testFolder/2017-2-5 testFile.md" ' --prune-empty --tag-name-filter cat -- --all
git push origin --force --all
git push origin --force --tags
git for-each-ref --format='delete %(refname)' refs/original | git update-ref --stdin
git reflog expire --expire=now --all
git gc --prune=nowgit count-objects -v
```