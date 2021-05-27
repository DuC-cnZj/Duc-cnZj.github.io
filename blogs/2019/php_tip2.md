---
title: 绕过爸爸调用爷爷的方法
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - php
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