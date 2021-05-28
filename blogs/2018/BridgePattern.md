---
title:  设计模式---桥接模式（Bridge Pattern）
date: "2018-10-29 14:49:43"
sidebar: false
categories:
 - 技术
tags:
 - 设计模式
 - 桥接模式
publish: true
---


## 概念

> ***桥接（Bridge）***是用于把抽象化与实现化解耦，使得二者可以独立变化。这种类型的设计模式属于结构型模式，它通过提供抽象化和实现化之间的桥接结构，来实现二者的解耦。
>
> 这种模式涉及到一个作为桥接的接口，使得实体类的功能独立于接口实现类。这两种类型的类可被结构化改变而互不影响。
>
> 我们通过下面的实例来演示桥接模式（Bridge Pattern）的用法。其中，可以使用相同的抽象类方法但是不同的桥接实现类，来画出不同颜色的圆。
>
> 
>
> ***大概就是两个相亲团队（interface），都有很多单身人士（implements），将位于这两个团队的单身狗一对一结合起来***



### 结构型设计模式：用于处理类和对象的组合



## 代码😀

```php
<?php

//抽象类：电脑
abstract class Computer{
    protected $phone;

    public function __construct($phone)
    {
        $this->phone = $phone;
    }
    public  function &__get($property_name)
    {
        if(isset($this->$property_name))
        {
            return($this->$property_name);
        }
        else
        {
            return(NULL);
        }
    }
    public  function __set($property_name, $value)
    {
        $this->$property_name = $value;
    }
    public abstract function connect();
}
//接口：手机
interface Phone{
    public function connectImpl();
}
//华硕品牌的电脑
class  ASUSComputer extends Computer{
    public function __construct($phone)
    {
        $this->phone=$phone;
    }
    public function connect()
    {
        echo "华硕电脑";
        $this->phone->connectImpl();
    }
}
//戴尔品牌的电脑
class DellComputer extends Computer{
    public function __construct($phone)
    {
        $this->phone=$phone;
    }
    public function connect()
    {
        echo "戴尔电脑";
        $this->phone->connectImpl();
    }

}
//三星手机
class  SamsungPhone implements Phone{
    public function connectImpl()
    {
        echo "连接了三星手机\n";
    }
}
//小米手机
class XiaomiPhone implements Phone{
    public function connectImpl()
    {
        echo "连接了小米手机\n";
    }
}
//抽象类：人
abstract class Person{
    public $computer;
    public function __construct($computer)
    {
        $this->computer = $computer;
    }
    public abstract function useComputer();
}
//学生
class Student extends Person{
    public function useComputer()
    {
        echo "学生使用";
        $this->computer->connect();
    }

}
 //老师
class Teacher extends Person{
    public function useComputer()
    {
        echo "教师使用";
        $this->computer->connect();
    }

}
function main(){
    //华硕电脑连接了小米手机
    $asusComputer=new ASUSComputer(new XiaomiPhone());
    $asusComputer->connect();
    //戴尔电脑连接了三星手机
    $dellComputer=new DellComputer(new SamsungPhone());
    $dellComputer->connect();
    //学生使用华硕电脑连接了小米手机
    $student=new Student(new ASUSComputer(new XiaomiPhone()));
    $student->useComputer();
    //教师使用戴尔电脑连接了三星手机
    $teacher=new Teacher(new DellComputer(new SamsungPhone()));
    $teacher->useComputer();
}
main();
```





## 介绍

> **意图：**将抽象部分与实现部分分离，使它们都可以独立的变化。
>
> **主要解决：**在有多种可能会变化的情况下，用继承会造成类爆炸问题，扩展起来不灵活。
>
> **何时使用：**实现系统可能有多个角度分类，每一种角度都可能变化。
>
> **如何解决：**把这种多角度分类分离出来，让它们独立变化，减少它们之间耦合。
>
> **关键代码：**抽象类依赖实现类。
>
> **应用实例：** 1、猪八戒从天蓬元帅转世投胎到猪，转世投胎的机制将尘世划分为两个等级，即：灵魂和肉体，前者相当于抽象化，后者相当于实现化。生灵通过功能的委派，调用肉体对象的功能，使得生灵可以动态地选择。 2、墙上的开关，可以看到的开关是抽象的，不用管里面具体怎么实现的。
>
> **优点：** 1、抽象和实现的分离。 2、优秀的扩展能力。 3、实现细节对客户透明。
>
> **缺点：**桥接模式的引入会增加系统的理解与设计难度，由于聚合关联关系建立在抽象层，要求开发者针对抽象进行设计与编程。
>
> **使用场景：** 1、如果一个系统需要在构件的抽象化角色和具体化角色之间增加更多的灵活性，避免在两个层次之间建立静态的继承联系，通过桥接模式可以使它们在抽象层建立一个关联关系。 2、对于那些不希望使用继承或因为多层次继承导致系统类的个数急剧增加的系统，桥接模式尤为适用。 3、一个类存在两个独立变化的维度，且这两个维度都需要进行扩展。
>
> **注意事项：**对于***两个独立变化***的维度，使用桥接模式再适合不过了。

