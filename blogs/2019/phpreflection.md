---
title: php 反射
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - php
tags:
 - 反射
 - php
publish: true
---


> https://www.php.net/manual/zh/book.reflection.php

```php
<?php


class ExampleTest extends TestCase {
    /**
     * @var ReflectionClass
     */
    protected $r;
    protected $c;

    protected function setUp (): void {
        parent::setUp();
        $this->c = $argument = new \App\User('duc', 23);
        $argument->notDefault = 111;
        $this->r = new ReflectionClass($argument);
        // 既可以是包含类名的字符串（string）也可以是对象（object）
    }

    public function testReflectionClass () {
        dump(['$this->r->getConstant()' => $this->r->getConstant('VERSION')]); // 获取常量没有就返回 false
        //        [
        //            '$this->r->getConstant()' => '1.0.0',
        //        ]
        dump(['$this->r->getConstants()' => $this->r->getConstants()]); // 获取常量数组
        //        [
        //            '$this->r->getConstants()' =>
        //                [
        //                    'VERSION' => '1.0.0',
        //                    'NAME'    => 'test',
        //                ],
        //        ]
        dump(['$this->r->getConstructor()' => $this->r->getConstructor()]);
        //        [
        //            '$this->r->getConstructor()' =>
        //                ReflectionMethod::__set_state([
        //                    'name'  => '__construct',
        //                    'class' => 'App\\User',
        //                ]),
        //        ]
        dump(['$this->r->getDefaultProperties()' => $this->r->getDefaultProperties()]);
        //        [
        //            '$this->r->getDefaultProperties()' =>
        //                [
        //                    'name' => 'default',
        //                    'age'  => NULL,
        //                ],
        //        ]
        dump(['$this->r->getDocComment()' => $this->r->getDocComment()]);
        //        [
        //            '$this->r->getDocComment()' => <<<DOC
        //            /**
        //             * Class User
        //             * @package App
        //             */'
        //DOC
        //        ]
        dump(['$this->r->getEndLine()' => $this->r->getEndLine()]);
        //        [
        //            '$this->r->getEndLine()' => 51,
        //        ]

        dump(['$this->r->getExtension()' => (New ReflectionClass('ReflectionClass'))->getExtension()]);
        // 如果是用户定义的类则返回 NULL。还不知道用在哪儿
        //        [
        //            '$this->r->getExtension()' =>
        //                ReflectionExtension::__set_state([
        //                    'name' => 'Reflection',
        //                ]),
        //        ]
        dump(['$this->r->getExtensionName()' => (New ReflectionClass('ReflectionClass'))->getExtensionName()]);
        //        [
        //            '$this->r->getExtensionName()' => 'Reflection',
        //        ]
        dump(['$this->r->getFileName()' => $this->r->getFileName()]);
        //        [
        //            '$this->r->getFileName()' => '/Users/congcong/duc/learn-reflect/src/User.php',
        //        ]
        dump(['$this->r->getInterfaceNames()' => $this->r->getInterfaceNames()]);
        //        [
        //            '$this->r->getInterfaceNames()' =>
        //                [
        //                    0 => 'App\\Imp',
        //                    1 => 'App\\Imp2',
        //                ],
        //        ]
        dump(['$this->r->getInterfaces()' => $this->r->getInterfaces()]);
        //        [
        //            '$this->r->getInterfaces()' =>
        //                [
        //                    'App\\Imp'  =>
        //                        ReflectionClass::__set_state([
        //                            'name' => 'App\\Imp',
        //                        ]),
        //                    'App\\Imp2' =>
        //                        ReflectionClass::__set_state([
        //                            'name' => 'App\\Imp2',
        //                        ]),
        //                ],
        //        ]
        dump(['$this->r->getMethod()' => $this->r->getMethod('getName')]);
        // 不存在就抛异常
        //        [
        //            '$this->r->getMethod()' =>
        //                ReflectionMethod::__set_state([
        //                    'name'  => 'getName',
        //                    'class' => 'App\\User',
        //                ]),
        //        ]
        dump(['$this->r->getMethods()' => $this->r->getMethods()]);
        // 私有公有都可以拿出
        //        [
        //            '$this->r->getMethods()' =>
        //                [
        //                    0 =>
        //                        ReflectionMethod::__set_state([
        //                            'name'  => '__construct',
        //                            'class' => 'App\\User',
        //                        ]),
        //                    1 =>
        //                        ReflectionMethod::__set_state([
        //                            'name'  => 'getName',
        //                            'class' => 'App\\User',
        //                        ]),
        //                    2 =>
        //                        ReflectionMethod::__set_state([
        //                            'name'  => 'setName',
        //                            'class' => 'App\\User',
        //                        ]),
        //                ],
        //        ]
        dump(['$this->r->getModifiers()' => $this->r->getModifiers()]);
        // 不知道这么用
        dump(['$this->r->getName()' => $this->r->getName()]);
        //        [
        //            '$this->r->getName()' => 'App\\User',
        //        ]
        dump(['$this->r->getNamespaceName()' => $this->r->getNamespaceName()]);
        //        [
        //            '$this->r->getNamespaceName()' => 'App',
        //        ]
        dump(['$this->r->getParentClass()' => $this->r->getParentClass()]);
        //        [
        //            '$this->r->getParentClass()' =>
        //                ReflectionClass::__set_state([
        //                    'name' => 'App\\Base',
        //                ]),
        //        ]

        dump(['$this->r->getProperties()' => $this->r->getProperties()]);
        //        [
        //            '$this->r->getProperties()' =>
        //                [
        //                    0 =>
        //                        ReflectionProperty::__set_state([
        //                            'name'  => 'name',
        //                            'class' => 'App\\User',
        //                        ]),
        //                    1 =>
        //                        ReflectionProperty::__set_state([
        //                            'name'  => 'age',
        //                            'class' => 'App\\User',
        //                        ]),
        //                ],
        //        ]

        dump(['$this->r->getProperty()' => $this->r->getProperty('name')]);
        //        [
        //            '$this->r->getProperty()' =>
        //                ReflectionProperty::__set_state([
        //                    'name'  => 'name',
        //                    'class' => 'App\\User',
        //                ]),
        //        ]
        dump(['$this->r->getReflectionConstant()' => $this->r->getReflectionConstant('VERSION')]);
        // 返回 ReflectionClassConstant 对象
        //        [
        //            '$this->r->getReflectionConstant()' =>
        //                ReflectionClassConstant::__set_state([
        //                    'name'  => 'VERSION',
        //                    'class' => 'App\\User',
        //                ]),
        //        ]
        dump(['$this->r->getReflectionConstants()' => $this->r->getReflectionConstants()]);
        //        [
        //            '$this->r->getReflectionConstants()' =>
        //                [
        //                    0 =>
        //                        ReflectionClassConstant::__set_state([
        //                            'name'  => 'VERSION',
        //                            'class' => 'App\\User',
        //                        ]),
        //                    1 =>
        //                        ReflectionClassConstant::__set_state([
        //                            'name'  => 'NAME',
        //                            'class' => 'App\\User',
        //                        ]),
        //                ],
        //        ]

        dump(['$this->r->getShortName()' => $this->r->getShortName()]);
        //        [
        //            '$this->r->getShortName()' => 'User',
        //        ]
        dump(['$this->r->getStartLine()' => $this->r->getStartLine()]);
        //        class User extends Base implements Imp, Imp2 {
        //        [
        //            '$this->r->getStartLine()' => 9,
        //        ]
        dump(['$this->r->getStaticProperties()' => $this->r->getStaticProperties()]);
        //        [
        //            '$this->r->getStaticProperties()' =>
        //                [
        //                    'app' => 'app',
        //                ],
        //        ]
        dump(['$this->r->getStaticPropertyValue()' => $this->r->getStaticPropertyValue('app')]);
        //        [
        //            '$this->r->getStaticPropertyValue()' => 'app',
        //        ]
        dump(['$this->r->getTraitAliases()' => $this->r->getTraitAliases()]);
        //        [
        //            '$this->r->getTraitAliases()' =>
        //                [
        //                    'tGet' => 'App\\MyTrait::traitGet',
        //                ],
        //        ]
        dump(['$this->r->getTraitNames()' => $this->r->getTraitNames()]);
        //        [
        //            '$this->r->getTraitNames()' =>
        //                [
        //                    0 => 'App\\MyTrait',
        //                ],
        //        ]

        dump(['$this->r->getTraits()' => $this->r->getTraits()]);
        //        [
        //            '$this->r->getTraits()' =>
        //                [
        //                    'App\\MyTrait' =>
        //                        ReflectionClass::__set_state([
        //                            'name' => 'App\\MyTrait',
        //                        ]),
        //                ],
        //        ]
        dump(['$this->r->hasConstant()' => $this->r->hasConstant('VERSION')]);
        // true
        dump(['$this->r->hasMethod()' => $this->r->hasMethod("notExits")]);
        // false
        dump(['$this->r->hasProperty()' => $this->r->hasProperty('app')]);
        // true
        dump(['$this->r->implementsInterface()' => $this->r->implementsInterface(\App\Imp::class)]);
        // true
        dump(['$this->r->inNamespace()' => $this->r->inNamespace()]);
        //  检查是否位于命名空间中
        dump(['$this->r->isAbstract()' => $this->r->isAbstract()]);
        // false
        dump(['$this->r->isAnonymous()' => $this->r->isAnonymous()]);
        // false
        dump(['$this->r->isCloneable()' => $this->r->isCloneable()]);
        // false
        dump(['$this->r->isFinal()' => $this->r->isFinal()]);
        // false
        dump(['$this->r->isInstance()' => $this->r->isInstance(new \App\User('name', 1))]);
        // true 是不是 User 实例
        dump(['$this->r->isInstantiable()' => $this->r->isInstantiable()]);
        // 检查类是否可实例化
        dump(['$this->r->isInterface()' => $this->r->isInterface()]);
        // false

        dump(['$this->r->isInternal()' => $this->r->isInternal()]);
        //  检查类是否由扩展或核心在内部定义
        dump(['$this->r->isIterable()' => $this->r->isIterable()]);
        // true
        dump(['$this->r->isIterateable()' => $this->r->isIterable()]);
        // 不清楚
        dump(['$this->r->isSubclassOf()' => $this->r->isSubclassOf(\App\BaseBase::class)]);
        // true
        dump(['$this->r->isTrait()' => $this->r->isTrait()]);
        // false
        dump(['$this->r->isUserDefined()' => $this->r->isUserDefined()]);
        // true
        dump(['$this->r->newInstance()' => $in = $this->r->newInstance("name", 11)]);
        $this->assertInstanceOf(\App\User::class, $in);
        dump(['$this->r->newInstanceArgs()' => $this->r->newInstanceArgs(['duc', 1])]);
        // 按顺序
        //        [
        //            '$this->r->newInstanceArgs()' =>
        //                App\User::__set_state([
        //                    'name' => 'duc',
        //                    'age'  => 1,
        //                ]),
        //        ]
        dump(['$this->r->newInstanceWithoutConstructor()' => $this->r->newInstanceWithoutConstructor()]);
        //        [
        //            '$this->r->newInstanceWithoutConstructor()' =>
        //                App\User::__set_state([
        //                    'name' => 'default',
        //                    'age'  => NULL,
        //                ]),
        //        ]

        dump(['$this->r->setStaticPropertyValue()' => $this->r->setStaticPropertyValue('app', 'apppp')]);
        dump(($this->c)::$app);
        // apppp
    }

    public function testReflectionClassConstant () {
        $c = $this->r->getReflectionConstant('VERSION');
        $this->assertTrue($c->isPublic());
        $this->assertFalse($c->isPrivate());
        $this->assertFalse($c->isProtected());
        $this->assertEquals('VERSION', $c->getName());
        $this->assertEquals(\App\User::class, $c->getDeclaringClass()->getName());
        $this->assertEquals('1.0.0', $c->getValue());
        echo $c; // Constant [ public string VERSION ] { 1.0.0 }
    }

    public function testReflectionFunction () {
        $fun = new ReflectionFunction('dd');
        $this->assertInstanceOf(Closure::class, $fun->getClosure());
        //        $fun->invoke(1,1,2);
        //        $fun->invokeArgs([11,22]);
        dd($fun->isDisabled());//false
    }

    public function testReflectionMethod () {
        $m = $this->r->getMethod('getName');
        dump($m->getDeclaringClass());
        //        class ReflectionClass#14 (1) {
        //           public $name =>
        //           string(8) "App\User"
        //        }

        $this->assertEquals('duc', $m->invoke($this->c));

        $this->assertTrue($m->isPublic());
        $this->assertFalse($m->isPrivate());
        $this->assertFalse($m->isProtected());
        $this->assertFalse($m->isAbstract());
        $this->assertFalse($m->isConstructor());
        $this->assertFalse($m->isDestructor());
        $this->assertFalse($m->isFinal());
        $this->assertFalse($m->isStatic());
    }

    public function testReflectionParameter () {
        $p = $this->r->getConstructor()->getParameters();
        dump($p[0]->allowsNull());// false
        dump($p[0]->canBePassedByValue());// false
        dump($p[0]->getDeclaringClass());
        //        ReflectionClass::__set_state([
        //            'name' => 'App\\User',
        //        ])
        dump($p[0]->getDeclaringFunction());
        //        ReflectionMethod::__set_state(array(
        //            'name' => '__construct',
        //            'class' => 'App\\User',
        //        ))
        dump($p[1]->getDefaultValue());
        // 10 ,没有会报错

        dump($p[1]->getDefaultValueConstantName());
        // null
        dump($this->r->getMethod('duc')->getParameters()[0]->getDefaultValueConstantName());
        // App\\PHP_INT_MIN
        dump($p[1]->getName());
        // age
        dump($p[1]->getPosition());
        // 1
        dump($p[0]->getPosition());
        // 0
        dump($p[0]->getType());
        //        ReflectionNamedType::__set_state([
        //        ])
        dump($p[0]->hasType());
        // true
        dump($p[0]->isCallable());
        // false
        dump($p[1]->isDefaultValueAvailable());
        // true
        dump($p[1]->isOptional());
        // true
        dump($p[1]->isVariadic());
        // false
        dump($this->r->getMethod('duc')->getParameters()[0]->isDefaultValueConstant());
        // true
    }

    public function test6 () {
        $pr = $this->r->getProperty('app');
        dump($pr->getName());
        // app
        dump($pr->getValue());
        // app

        dump($pr->isDefault());
        // true
        dump($pr->isStatic());
        // true
    }

    public function test7 () {
        $r = $this->r->getMethods(ReflectionMethod::IS_PRIVATE);
        $r[0]->setAccessible(true);
        $r[0]->invoke($this->c, '改TM的');
        $this->assertEquals('改TM的', $this->c->getName());
        foreach ($this->r->getProperties(ReflectionProperty::IS_PRIVATE) as $property) {
            $property->setAccessible(true);
            dump($property->getValue($this->c));
        }
    }

    public function test8 () {
        $r = new ReflectionClass(\App\User::class);
        // 既可以是包含类名的字符串（string）也可以是对象（object）
        dump($r->getProperties());
        dump($r->getMethod('getName')->invoke(new \App\User("名字",1)));
        dump($r->getMethod('getName')->getDocComment());
//        /**
//         * @return string
//         *
//         * @author 神符 <1025434218@qq.com>
//         */'
    }
}
```


