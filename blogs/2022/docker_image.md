---
title: docker 镜像版本的含义
date: '2022-01-25 15:20:38'
sidebar: false
categories:
 - 技术
tags:
 - docker
publish: true
---

# docker 镜像版本的含义

[docker镜像Alpine, Slim, Stretch, Buster, Jessie, Bullseye, windowsservercore区别 - 提米果的博客](https://www.timiguo.com/archives/223/)

经常看到docker image 有这些tag Alpine, Slim, Stretch, Buster, Jessie, Bullseye,  
之前一直知道最小选择Alpine，想稳定就选Stretch，一直没探究他们的区别，现在总结一下

## stretch/buster/jessie

稳定的[Debian](https://wiki.debian.org/DebianReleases)发行版是10.7（2020-12-05），其代号是Buster。Stretch是所有版本9变体的代号，Jessie是所有版本8变体的代号。正在开发但尚未稳定的未来版本是Bullseye和Bookworm和Trixie。现在在一些新的DockerHub的映像tag列表中常常看到这些tag。  
如果您的代码与Debian操作系统的特定版本兼容，请选择这些映像之一,但如果是做新项目，明确清楚程序里没依赖老系统的api，则用最新稳定版的tag.

## slim

仅安装运行特定工具所需的最少软件包,如果有空间限制并且不需要完整版本，请使用此tag,但是使用前需要经过完整测试，如果没时间测试，就使用上面的完整版本

## alpine

Alpine映像基于[Alpine Linux](https://alpinelinux.org/)专门为在容器内部使用而构建的操作系统，占用空间最少，但是出现问题的时候比较难调试，因为什么工具都没有，如果优先考虑空间大小，肯定选这种tag.缺点是它不包含你可能需要的某些软件包。主要是，它使用更小的musl-lib代替glibc。如果你的程序需要的glibc特性功能，那就别用他。

## windowsservercore

一直没用过，而且镜像都是上G的，特别大，如果真要跑iis等微软系统才能跑等软件才去用他吧

## 选择套路

- 没时间测试程序对系统对依赖和兼容性的话，直接用stretch/buster/jessie这些准没错
- 有时间测试程序对系统对依赖和兼容性的话，可以选择slim和alpine，像golang，如果使用了cgo,一定不要选择alpine。当然如果要求空间极致，肯定优先选择alpine
- 在k8s集群中，选取镜像最好是和主机os一致的分发版本
- 依赖windows系统的没得选，只能选windowsservercore

## 套路总结

时间换空间?空间换时间?
