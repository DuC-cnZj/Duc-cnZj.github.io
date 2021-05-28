---
title: 绕过爸爸调用爷爷的方法
date: '2019-10-09 13:20:44'
sidebar: false
categories:
 - 技术
tags:
 - tip
 - php
publish: true
---



```php
<?php

class Anchestor {
  
   public $Prefix = '';

   private $_string =  'Bar';
    public function Foo() {
        return $this->Prefix.$this->_string;
    }
}

class MyParent extends Anchestor {
    public function Foo() {
         $this->Prefix = null;
        return parent::Foo().'Baz';
    }
}

class Child extends MyParent {
    public function Foo() {
        $this->Prefix = 'Foo';
        return Anchestor::Foo();
    }
}

$c = new Child();
echo $c->Foo(); //return FooBar, because Prefix, as in Anchestor::Foo()
```