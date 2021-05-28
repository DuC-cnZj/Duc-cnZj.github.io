---
title:  nginx 出现413 Request Entity Too Large问题的解决方法
date: "2018-11-09 11:59:37"
sidebar: false
categories:
 - 技术
tags:
 - bug
 - nginx
publish: true
---


> - 使用php上传图片（大小1.9M)，出现 nginx: 413 Request Entity Too Large 错误。
> - 根据经验是服务器限制了上传文件的大小，但php默认的文件上传是2M，应该不会出现问题。
> - 打开php.ini，把 upload_max_filesize 和 post_max_size 修改为20M，然后重启。
> - 再次上传，问题依旧，可以排除php方面的问题。原来nginx默认上传文件的大小是1M，可nginx的设置中修改。   ` client_max_body_size 20M`


![2018_11_09_MY5xLpiD9T.gif](../images/2018_11_09_MY5xLpiD9T.gif)