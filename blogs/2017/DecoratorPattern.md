---
title:  设计模式---装饰器模式（Decorator Pattern）
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - 设计模式
tags:
 - 装饰器模式
publish: true
---

## 概念

> 允许向一个现有的对象添加新的功能，同时又不改变其结构。



### 结构型设计模式：用于处理类和对象的组合



## &#x1f604; 代码

```Php
<?php

interface Color
{
	public function getColor();

	public function colorInfo();
}

class White implements Color
{
	public function getColor()
	{
		return '白色';
	}

	public function colorInfo()
	{
		return '这是一张白纸';
	}
}

class Red implements Color
{
	public function __construct(Color $color)
	{
		$this->color = $color;
	}

	public function getColor()
	{
		return '红色' . $this->color->getColor();		
	}

	public function colorInfo()
	{
		return $this->color->colorInfo() . '加了点红';
	}
}

class Green implements Color
{
	public function __construct(Color $color)
	{
		$this->color = $color;
	}

	public function getColor()
	{
		return '绿色' . $this->color->getColor();		
	}

	public function colorInfo()
	{
		return $this->color->colorInfo() . '加了点绿';
	}
}

// 基础颜色白色，在白色的基础上去装饰
$color = new Red(new Green(new White));
// echo $color->getColor();
echo $color->colorInfo();
```





## 介绍

> **意图：**动态地给一个对象添加一些额外的职责。就增加功能来说，装饰器模式相比生成子类更为灵活。
>
> **主要解决：**一般的，我们为了扩展一个类经常使用继承方式实现，由于继承为类引入静态特征，并且随着扩展功能的增多，子类会很膨胀。
>
> **何时使用：**在不想增加很多子类的情况下扩展类。
>
> **如何解决：**将具体功能职责划分，同时继承装饰者模式。
>
> **关键代码：** 1、Component 类充当抽象角色，不应该具体实现。 2、修饰类引用和继承 Component 类，具体扩展类重写父类方法。
>
> **应用实例：** 1、孙悟空有 72 变，当他变成"庙宇"后，他的根本还是一只猴子，但是他又有了庙宇的功能。 2、不论一幅画有没有画框都可以挂在墙上，但是通常都是有画框的，并且实际上是画框被挂在墙上。在挂在墙上之前，画可以被蒙上玻璃，装到框子里；这时画、玻璃和画框形成了一个物体。
>
> **优点：**装饰类和被装饰类可以独立发展，不会相互耦合，装饰模式是继承的一个替代模式，装饰模式可以动态扩展一个实现类的功能。
>
> **缺点：**多层装饰比较复杂。
>
> **使用场景：** 1、扩展一个类的功能。 2、动态增加功能，动态撤销。
>
> **注意事项：**可代替继承。





## 在 laravel 中实践，可以用来做缓存层

### ViewImp.php

```php
<?php

namespace App\Services;


interface ViewImp
{
    public function get();
}
```

### View.php

```php
<?php

namespace App\Services;

use App\User;

class View implements ViewImp
{
    public function get()
    {
        return User::all();
    }
}
```

### ViewCache.php

```php
<?php


namespace App\Services;


use Illuminate\Support\Facades\Cache;

class ViewCache implements ViewImp
{
    /**
     * @var ViewImp
     */
    private $imp;

    public function __construct(ViewImp $imp)
    {
        $this->imp = $imp;
    }

    public function get()
    {
        return Cache::remember('users', 60, function () {
            echo "没有使用缓存";
            return $this->imp->get();
        });
    }
}
```

### AppServiceProvider.php

```php
public function register()
{
    $this->app->singleton(ViewImp::class, function () {
        return new ViewCache(new View());
    });
}
```

### web.php

```php
Route::get('/', function (\App\Services\ViewImp $imp) {
    return $imp->get();
});
```

