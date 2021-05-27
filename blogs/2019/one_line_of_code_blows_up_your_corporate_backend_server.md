---
title: 内存泄漏
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - 内存泄漏
tags:
 - 内存泄漏
publish: true
---


> swoole 等常驻内存，在请求结束时不会自动销毁
>
> - 使用`global`关键词声明的变量
> - 使用`static`关键词声明的类静态变量、函数静态变量
> - `PHP`的超全局变量，包括`$_GET`、`$_POST`、`$GLOBALS`等
>
> 这些变量将一直存在内存中，如果使用不当那么将造成内存泄漏，下面是例子
>
> <https://wiki.swoole.com/wiki/page/p-zend_mm.html>



如果你公司使用swoole等常驻内存，那么你加下面这行代码就能让你公司的后端服务器每隔一段时间就出现500错误。😘

```php
$GLOBALS['name'] .= str_repeat(1, 100000);
```

下面的例子也可以

```php
<?php

class Test
{
    static $array = array();
    static $string = '';
}

function onReceive($serv, $fd, $reactorId, $data)
{
    Test::$array[] = $fd;
    Test::$string .= $data;
}
```
