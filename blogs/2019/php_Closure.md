---
title: php Closure 理解
date: '2019-10-15 13:32:45'
sidebar: false
categories:
 - 技术
tags:
 - Closure
 - php
publish: true
---

```php
<?php


class ClosureTest extends TestCase {
    public function test1 () {

        $cl1 = static function () {
            return A::$sfoo;
        };
        $cl2 = function () {
            return $this->ifoo;
        };

        $bcl1 = Closure::bind($cl1, null, 'A');
        $bcl2 = Closure::bind($cl2, new A(), "A");

        echo $bcl1(), "\n";
        echo $bcl2(), "\n";
    }

    public function test2 () {
        (function () {
            dump($this->ifoo);
        })->bindTo(new A(), 'A')();
        (function () {
            dump(A::$sfoo);
        })->bindTo(null, 'A')();
    }
}

class A {
    private static $sfoo = 1;
    protected $ifoo = 2;
}

```

## `Closure::bind()` 和 `闭包->bindTo()` 是一个东西

```php
$closure  = function () {
  echo $this->name;
};

class A {
  public $name = 'duc';
}

$closure->bindTo(new A);
# eq
\Closure::bind($closure, new A);

```

## 作用域

> `newScope` 参数如果加了，就好比调用者就是 `newScope` 这个类，这个类能访问所有的属性，
>
> 如果不加默认就是 static，有种根据上下文的感觉，下面这样的调用就好比别人调用你这个类，所以只能访问公有属性

```php
$closure  = function () {
  echo $this->name;
};

$closure1  = function () {
  echo $this->age;
};

$closure2  = function () {
  echo $this->fullName;
};

class A {
  private $name = 'duc';
  public $age = 23;
  protected $fullName = "sf";
}

$closure->bindTo(new A)(); # 报错
$closure->bindTo(new A, 'A')(); # duc

$closure1->bindTo(new A)(); # 23
$closure1->bindTo(new A, 'A')(); # 23


$closure2->bindTo(new A)(); # 报错
$closure2->bindTo(new A, 'A')(); # sf
```

```php
public function test4 () {
  $closure  = function () {
    echo $this->ifoo;
  };

  \Closure::bind($closure, new A)(); # 报错
  \Closure::bind($closure, new A, "C")(); // 2
  
  \Closure::bind($closure, new A, "C")(); // 2 # 数值以实例(A)的为准
  \Closure::bind($closure, new C, "A")(); // 3 # 数值以实例(C)的为准
}

class A {
    private static $sfoo = 1;
    protected $ifoo = 2;
}

class C extends A {
	protected $ifoo = 3;
}
```

## 如何在不给类添加任何代码的情况下调用该类的私有方法和属性
```php
    public function testClosure () {
        $u = new Duc();
        $closure = function () {
          dump($this->getName());
        };

        $closure->bindTo($u, Duc::class)(); // default

        $closure1 = function () {
            $this->name = 'duc';
            dump($this->getName()); // duc
        };
        $closure1->bindTo($u, Duc::class)();
    }
		
class Duc
{
    private $name = 'default';

    private function getName () {
        return $this->name;
    }
}
```

## 把 newscope 当成调用者
```php

class C {
    private $name = 'C';
}

class E extends C {
    private $name = 'E';
}
public function test11()
{
		$c = function () {
				echo $this->name;
		};

		$objc = new C;
		$obje = new E;

		$c->bindTo($obje, C::class)(); // C
		$c->bindTo($obje, E::class)(); // E

		$c->bindTo($objc, C::class)(); // C
		$c->bindTo($objc, E::class)(); // ErrorException : Undefined property: Tests\Feature\E::$name
}
```