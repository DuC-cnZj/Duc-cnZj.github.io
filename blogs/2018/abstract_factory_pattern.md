---
title:  设计模式---抽象工厂模式（Abstract Factory Pattern）
date: "2018-10-30 15:42:18"
sidebar: false
categories:
 - 技术
tags:
 - 设计模式
 - 抽象工厂模式
publish: true
---



## 概念

> ***抽象工厂模式（Abstract Factory Pattern）***是围绕一个超级工厂创建其他工厂。该超级工厂又称为其他工厂的工厂。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
>
> 在抽象工厂模式中，接口是负责创建一个相关对象的工厂，不需要显式指定它们的类。每个生成的工厂都能按照工厂模式提供对象。



> 工厂方法针对每一种产品提供一个工厂类，通过不同的工厂实例来创建不同的产品实例，在同一等级结构中，支持增加任意产品。
>
> 抽象工厂是应对产品族概念的，比如说，每个汽车公司可能要同时生产轿车，货车，客车，那么每一个工厂都要有创建轿车，货车和客车的方法。应对产品族概念而生，增加新的产品线很容易，但是无法增加新的产品。





### 创建型设计模式：用于处理对象的实例化



## 代码😀

```php

<?php  
 /** 
 * 抽象工厂模式  
 */  
  
//抽象工厂  
interface AnimalFactory {  
      
    public function createCat();  
    public function createDog();  
      
}  
  
//具体工厂  
class BlackAnimalFactory implements AnimalFactory {  
      
    function createCat(){  
        return new BlackCat();  
    }  
      
    function createDog(){  
        return new BlackDog();    
    }  
}  
  
class WhiteAnimalFactory implements AnimalFactory {  
      
    function createCat(){  
        return new WhiteCat();  
    }  
      
    function createDog(){  
        return new WhiteDog();  
    }  
}  
  
//抽象产品  
interface Cat {  
    function Voice();  
}  
  
interface Dog {  
    function Voice();     
}  
  
//具体产品  
class BlackCat implements Cat {  
      
    function Voice(){  
        echo '黑猫喵喵……';  
    }  
}  
  
class WhiteCat implements Cat {  
      
    function Voice(){  
        echo '白猫喵喵……';  
    }  
}  
  
class BlackDog implements Dog {  
      
    function Voice(){  
        echo '黑狗汪汪……';        
    }  
}  
  
class WhiteDog implements Dog {  
      
    function Voice(){  
        echo '白狗汪汪……';        
    }  
}  
  
//客户端  
class Client {  
      
    public static function main() {  
        self::run(new BlackAnimalFactory());  
        self::run(new WhiteAnimalFactory());  
    }  
      
    public static function run(AnimalFactory $AnimalFactory){  
        $cat = $AnimalFactory->createCat();  
        $cat->Voice();  
          
        $dog = $AnimalFactory->createDog();  
        $dog->Voice();  
    }  
}  
Client::main(); 
```



## 介绍

> ***意图：***提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。
>
> ***主要解决***：主要解决接口选择的问题。
>
> ***何时使用：***系统的产品有多于一个的产品族，而系统只消费其中某一族的产品。
>
> ***如何解决：***在一个产品族里面，定义多个产品。
>
> ***关键代码：***在一个工厂里聚合多个同类产品。
>
> ***应用实例：***工作了，为了参加一些聚会，肯定有两套或多套衣服吧，比如说有商务装（成套，一系列具体产品）、时尚装（成套，一系列具体产品），甚至对于一个家庭来说，可能有商务女装、商务男装、时尚女装、时尚男装，这些也都是成套的，即一系列具体产品。假设一种情况（现实中是不存在的，要不然，没法进入共产主义了，但有利于说明抽象工厂模式），在您的家中，某一个衣柜（具体工厂）只能存放某一种这样的衣服（成套，一系列具体产品），每次拿这种成套的衣服时也自然要从这个衣柜中取出了。用 OO 的思想去理解，所有的衣柜（具体工厂）都是衣柜类的（抽象工厂）某一个，而每一件成套的衣服又包括具体的上衣（某一具体产品），裤子（某一具体产品），这些具体的上衣其实也都是上衣（抽象产品），具体的裤子也都是裤子（另一个抽象产品）。
>
> ***优点：***当一个产品族中的多个对象被设计成一起工作时，它能保证客户端始终只使用同一个产品族中的对象。
>
> ***缺点：***产品族扩展非常困难，要增加一个系列的某一产品，既要在抽象的 Creator 里加代码，又要在具体的里面加代码。
>
> ***使用场景：*** 1、QQ 换皮肤，一整套一起换。 2、生成不同操作系统的程序。
>
> ***注意事项：***产品族难扩展，产品等级易扩展。



