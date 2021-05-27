---
title: php tip
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - php
tags:
 - tip
 - php
publish: true
---


### 差别，传参是不是数组

```php
# call_user_func
# call_user_func_array

call_user_func([AA::class, 'getName'], 1, 2);
call_user_func_array([AA::class, 'getName'], [1, 2]);
```



### callback


```
[AA::class, 'getName']
[new AA(), 'getName']
```



### 静态方法调用

```
(new AA())::getName()
AA::getName()
```



###  forward_static_call vs call_user_func

>
>  https://stackoverflow.com/questions/5061476/php-forward-static-call-vs-call-user-func



```php
<?php

namespace Tests\Unit;

use Tests\TestCase;

class DucTest extends TestCase
{
    public function test()
    {
        call_user_func([AA::class, 'getName'], 1, 2);
        call_user_func_array([AA::class, 'getName'], [1, 2]);
    }

    public function test1()
    {
        call_user_func([AA::class, 'getName'], 1, 2);
        call_user_func([new AA(), 'getName'], 1, 2);
        call_user_func_array([new AA(), 'getAge'], [1, 2]);
    }

    public function test2()
    {
        call_user_func([BB::class, 'getName'], 1, 2);
        call_user_func([new BB(), 'parent::getAge'], 1, 2);

        call_user_func_array([new BB(), 'getAge'], [1, 2]);
        call_user_func_array([new BB(), 'parent::getAge'], [1, 2]);
    }

    public function test3()
    {
        // 静态方法调用
        AA::getName(1, 2);
        (new AA)::getName(1, 2);
    }

    public function test4()
    {
        forward_static_call([AA::class, 'getName'], 1, 2);
        forward_static_call_array([new AA(), 'getAge'], [1, 2]);
        forward_static_call_array('Tests\Unit\getName', [1, 2]);
        call_user_func('Tests\Unit\getName', 1, 2);
    }
}
// forward_static_call vs call_user_func
// https://stackoverflow.com/questions/5061476/php-forward-static-call-vs-call-user-func

class AA
{
    public static function getName($name, $user = null)
    {
        echo get_called_class() . '=>';
        echo 'AA::getName->' . $name . '->' . $user . "\n";

        return 1;
    }

    public function getAge($name, $user = null)
    {
        echo 'AA::getAge->' . $name . '->' . $user . "\n";
    }
}

class BB extends AA
{
    public static function getName($name, $user = null)
    {
        echo get_called_class();
        echo 'BB::getName->' . $name . '->' . $user . "\n";
    }

    public function getAge($name, $user = null)
    {
        echo 'BB::getAge->' . $name . '->' . $user . "\n";
    }
}

function getName($name, $user = null)
{
    echo 'Out getName->' . $name . '->' . $user . "\n";
}

```


### 析构函数就算抛异常也会执行

## defer

> 用php 实现go defer https://github.com/php-defer/php-defer

```php

class Defer
{
    public $defer = [];

    public function push($closure)
    {
        $this->defer[] = $this->getClosure($closure);

        return $this;
    }

    private function getClosure($closure)
    {
        if (is_callable($closure)) {
            return $closure;
        }

        return function () use ($closure) {
            return $closure;
        };
    }

    public function __destruct()
    {
        foreach (array_reverse($this->defer) as $closure) {
            $closure();
        }
    }
}
```

```php
$a = now();
// 注意这里不能内联调用 https://www.php.net/manual/zh/language.oop5.decon.php#105368
$d = new Defer();
$d->push(function () use ($a) {
	echo now()->since($a);
});

sleep(3);

// 3 seconds after

// 或者抽成一个方法, 注意这里必须要保证 Defer 有引用，不然会立即执行析构函数
public function testtt () {
	$t = now();
	$_ = $this->calTime($t);
	echo "b\n";
	sleep(3);
}

function calTime ($t) {
	$d = new Defer();

	$d->push(function () use ($t) {
			echo now()->since($t)."\n";
	});

	return $d;
}
```

```php

class Logger
{
    protected $rows = [];

    public function __construct()
    {
        register_shutdown_function([$this, 'dc']);
        register_shutdown_function([$this, 'adc']);
    }

    public function adc()
    {
        echo 'abc--';
    }
    public function dc()
    {
        echo 'dc--';
    }

    public function __destruct()
    {
        echo 'd';
    }

    public function log($row)
    {
        $this->rows[] = $row;
    }

    public function save()
    {
        echo '<ul>';
        foreach ($this->rows as $row) {
            echo '<li>', $row, '</li>';
        }
        echo '</ul>';
    }
}
```