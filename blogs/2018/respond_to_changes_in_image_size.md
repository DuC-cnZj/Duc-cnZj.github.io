---
title: 响应式的改变图片尺寸
date: "2018-12-07 16:30:17"
sidebar: false
categories:
 - 技术
tags:
 - 响应式
publish: true
---


> 在学习 [spatie/laravel-medialibrary](https://docs.spatie.be/laravel-medialibrary/v7/introduction) 扩展包的时候发现👇

原来html中有一个叫 `srcset` 的属性，能够响应式的改变图片尺寸

详情请看 [响应式图片](https://developer.mozilla.org/zh-CN/docs/Learn/HTML/Multimedia_and_embedding/Responsive_images)

以下是扩展包中的实现，很有用，可以用在博客上

> 这个package 能够对图片进行响应式的操作，把你上传的图片根据像素保存成多份，分别在不同的屏幕尺寸下返回不同大小的图片

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Document</title>
    <meta name="viewport" content="width=device-width">
</head>
<body>
    <img 
        src="http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_2048_1889.png 2048w"
        srcset="
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_2048_1889.png 2048w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_1713_1580.png 1713w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_1433_1322.png 1433w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_1199_1106.png 1199w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_1003_925.png 1003w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_839_774.png 839w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_702_647.png 702w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_587_541.png 587w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_491_453.png 491w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_411_379.png 411w, 
            http://localhost:9999/media/39/responsive-images/carbon___medialibrary_original_344_317.png 344w, 
            data:image/svg+xml;base64,PCFET0NUWVBFIHN2ZyBQVUJMSUMgIi0vL1czQy8vRFREIFNWRyAxLjEvL0VOIiAiaHR0cDovL3d3dy53My5vcmcvR3JhcGhpY3MvU1ZHLzEuMS9EVEQvc3ZnMTEuZHRkIj4KPHN2ZyB2ZXJzaW9uPSIxLjEiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgeG1sbnM6eGxpbms9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkveGxpbmsiIHhtbDpzcGFjZT0icHJlc2VydmUiIHg9IjAiCiB5PSIwIiB2aWV3Qm94PSIwIDAgMjA0OCAxODkwIj4KCTxpbWFnZSB3aWR0aD0iMjA0OCIgaGVpZ2h0PSIxODkwIiB4bGluazpocmVmPSJkYXRhOmltYWdlL2pwZWc7YmFzZTY0LC85ai80QUFRU2taSlJnQUJBUUVBWUFCZ0FBRC8vZ0E3UTFKRlFWUlBVam9nWjJRdGFuQmxaeUIyTVM0d0lDaDFjMmx1WnlCSlNrY2dTbEJGUnlCMk9EQXBMQ0J4ZFdGc2FYUjVJRDBnT1RBSy85c0FRd0FEQWdJREFnSURBd01EQkFNREJBVUlCUVVFQkFVS0J3Y0dDQXdLREF3TENnc0xEUTRTRUEwT0VRNExDeEFXRUJFVEZCVVZGUXdQRnhnV0ZCZ1NGQlVVLzlzQVF3RURCQVFGQkFVSkJRVUpGQTBMRFJRVUZCUVVGQlFVRkJRVUZCUVVGQlFVRkJRVUZCUVVGQlFVRkJRVUZCUVVGQlFVRkJRVUZCUVVGQlFVRkJRVUZCUVUvOEFBRVFnQUhRQWZBd0VSQUFJUkFRTVJBZi9FQUI4QUFBRUZBUUVCQVFFQkFBQUFBQUFBQUFBQkFnTUVCUVlIQ0FrS0MvL0VBTFVRQUFJQkF3TUNCQU1GQlFRRUFBQUJmUUVDQXdBRUVRVVNJVEZCQmhOUllRY2ljUlF5Z1pHaENDTkNzY0VWVXRId0pETmljb0lKQ2hZWEdCa2FKU1luS0NrcU5EVTJOemc1T2tORVJVWkhTRWxLVTFSVlZsZFlXVnBqWkdWbVoyaHBhbk4wZFhaM2VIbDZnNFNGaG9lSWlZcVNrNVNWbHBlWW1acWlvNlNscHFlb3FhcXlzN1MxdHJlNHVickN3OFRGeHNmSXljclMwOVRWMXRmWTJkcmg0dVBrNWVibjZPbnE4Zkx6OVBYMjkvajUrdi9FQUI4QkFBTUJBUUVCQVFFQkFRRUFBQUFBQUFBQkFnTUVCUVlIQ0FrS0MvL0VBTFVSQUFJQkFnUUVBd1FIQlFRRUFBRUNkd0FCQWdNUkJBVWhNUVlTUVZFSFlYRVRJaktCQ0JSQ2thR3h3UWtqTTFMd0ZXSnkwUW9XSkRUaEpmRVhHQmthSmljb0tTbzFOamM0T1RwRFJFVkdSMGhKU2xOVVZWWlhXRmxhWTJSbFptZG9hV3B6ZEhWMmQzaDVlb0tEaElXR2g0aUppcEtUbEpXV2w1aVptcUtqcEtXbXA2aXBxckt6dExXMnQ3aTV1c0xEeE1YR3g4akp5dExUMU5YVzE5aloydUxqNU9YbTUranA2dkx6OVBYMjkvajUrdi9hQUF3REFRQUNFUU1SQUQ4QStPL2lKOGR2RWxuNGluaWh1U3FnOFVwVkdtVHl4U09RbC9hRjhWeC84dlpOSlZXeW94aklqLzRhTDhWZjgvUnArMGtYN05IZGZDajQ3K0pOVTFpV09lNExLRUpyU0UyMlJPbWtqeXY0bUVueFJkZld1YWZ4R3FWMGNWS0c3OUtsV0xza1JWWUhvdndYL3dDUTdOLzF6TmFRM001N0ZmNGpBbnhYZEErdFoxTFhGR1RTT1V1SVNVUElyQk96TlZycVp4VWp0V3QwQjZMOEdGSTF5YmovQUpabXRhYnV6T2V4N2Y0NCtEbWozZXZUU3RKSUdKN0N0cFUwMllLYnNjODN3UjBWdXNzdjVWUHNvbGUwWXcvQXpRei9BTXRKUHlwK3ppSHRHZGo4Ti9nN28rbjZuSThieUVsU09SVlJna3laVGJQLzJRPT0iPgoJPC9pbWFnZT4KPC9zdmc+ 32w"
        onload="this.onload=null;this.sizes=Math.ceil(this.getBoundingClientRect().width/window.innerWidth*100)+'vw';"
        sizes="1px"
    >
</body>
</html>

```

![2018_12_07_TGBCFCx9hz.gif](../images/2018_12_07_TGBCFCx9hz.gif)

可以看到在屏幕尺寸改变的时候，浏览器会去请求不同尺寸的图像

## 非常好的一个图片处理的包👍