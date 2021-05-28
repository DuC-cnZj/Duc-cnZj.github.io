---
title: stat failed 记一次laravel 图片上传报错
date: "2018-11-16 15:17:44"
sidebar: false
categories:
 - 技术
tags:
 - laravel
 - bug
publish: true
---


[解决方案](http://php.net/manual/zh/splfileinfo.getsize.php)

> **所以，解决这个问题就是要在 move 方法调用之前调用 SplFileInfo 中的方法。**

在使用 laravel 框架上传图片的时候，本地没有出现过报错，但是部署的时候出现了，错误信息为 SplFileInfo::getSize(): stat failed ...。

```php
$file->move($path, $file_name);
$model = \App\Models\File::create([
    'user_id' => auth('api')->id() ?? 0,
    'size' => $file->getSize(),
    'mine_type' => $file->getClientMimeType(),
    'original_name' => $file->getClientOriginalName(),
    'original_extension' => $file->getClientOriginalExtension(),
    'save_path' => $floder . '/' . $file_name
]);
```
> 错误原因是先调用了 move() 方法保存了图片，后调用 getSize() 方法获取文件大小，为什么会出现这种错误，php官方文档 上已经有人做了比较好的解释。如果你正在使用 Symfony's UploadedFile，在调用了 move 方法之后再调用 SplFileInfo::getSize 要当心，
> 你很可能会得到一个难以发现的错误：`stat failed`如果你仔细思考，你会发现这是合理的。上传的文件已经被 Symfony 移动走了，
> 但是 getSize 方法属于 SplFileInfo ，然而 SplFileInfo 并不知道文件被移走了。奇怪的是，这个问题并没有出现在我的 mac 上。
