---
title: 用 Mockery mock 数据
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - php
tags:
 - mockery
 - php
publish: true
---


```php
<?php

namespace App\MockTest;

use Mockery\Undefined;
use PHPUnit\Framework\TestCase;

class One extends TestCase {
    public function testMock () {
        $m = \Mockery::mock(A::class);

        // 用法一，直接后面跟方法
        $m->allows()->get()->andReturn(true);
        $this->assertTrue($m->get());

        // 用法二
        $m->allows('getName')->andReturn('duc');
        $this->assertEquals('duc', $m->getName());

        // 用法三, 方法名 => 返回值
        $m->allows(['getAge' => 18, 'getMoney' => null]);
        $this->assertEquals(18, $m->getAge());
        $this->assertNull($m->getMoney());

        \Mockery::close();
    }

    public function testMock1 () {
        $m = \Mockery::mock(A::class);

        // 用法一，直接后面跟方法
        $m->shouldReceive()->get()->andReturn(true);
        $this->assertTrue($m->get());

        // 用法二
        $m->shouldReceive('getName')->andReturn('duc');
        $this->assertEquals('duc', $m->getName());

        // 用法三, 方法名 => 返回值
        $m->shouldReceive(['getAge' => 18, 'getMoney' => null]);
        $this->assertEquals(18, $m->getAge());
        $this->assertNull($m->getMoney());

        \Mockery::close();
    }

    public function testMock2 () {
        $m = \Mockery::mock(A::class);

        // 允许模拟不存在的方法
        $m->shouldIgnoreMissing();
        $this->assertNull($m->hi());

        // 允许模拟不存在的方法, 设置默认返回
        $m->shouldIgnoreMissing('duc');
        $this->assertEquals('duc', $m->hi());
        $this->assertEquals('duc', $m->bye());

        \Mockery::close();
    }

    public function testMock3 () {
        $m = \Mockery::mock(A::class);

        // 期望 getAge 方法被调用一次，多调用会报错
        $m->expects()->getAge()->andReturn(18);
        $this->assertEquals(18, $m->getAge());

        $this->expectException(\Mockery\Exception\InvalidCountException::class);
        $m->getAge();
        \Mockery::close();
    }

    public function testMock4 () {
        $m = \Mockery::mock(A::class);

        // 期望 getAge 方法被调用一次，多调用会报错
        $m->expects('getAge')->andReturn(18);
        $this->assertEquals(18, $m->getAge());

        $this->expectException(\Mockery\Exception\InvalidCountException::class);
        $m->getAge();

        \Mockery::close();
    }

    /**
     * @runInSeparateProcess
     * @preserveGlobalState disabled
     */
    public function testMock5 () {
        // 不允许模拟不存在的方法
        \Mockery::getConfiguration()->allowMockingNonExistentMethods(false);

        $m = \Mockery::mock(B::class);

        $m->shouldReceive('getName')->andReturn('aaa');
        $this->assertEquals('aaa', $m->getName());

        $this->expectException(\Mockery\Exception::class);
        $m->shouldReceive('getAge')->andReturn(18);

        \Mockery::close();
    }

    /**
     * @runInSeparateProcess
     * @preserveGlobalState disabled
     */
    public function testMock5x () {
        // 不允许模拟不存在的方法
        \Mockery::getConfiguration()->allowMockingNonExistentMethods(false);

        $m = \Mockery::mock(B::class);

        $m->shouldReceive('getName')->andReturn('aaa');
        $this->assertEquals('aaa', $m->getName());

        $m->shouldAllowMockingMethod('getAge');
        $m->shouldReceive('getAge')->andReturn(18);
        $this->assertEquals(18, $m->getAge());

        \Mockery::close();
    }

    public function testMock6 () {
        $m = \Mockery::mock(B::class);

        $m->shouldReceive('getName')->andReturn('aaa');
        $this->assertEquals('aaa', $m->getName());

        $this->expectException(\Mockery\Exception\BadMethodCallException::class);
        $this->assertEquals('x', $m->getX());

        \Mockery::close();
    }

    public function testMock7 () {
        $m = \Mockery::mock(B::class);

        // 不模拟的方法按原来的走
        $m->makePartial();

        $m->shouldReceive('getName')->andReturn('aaa');
        $this->assertEquals('aaa', $m->getName());

        $this->assertEquals('x', $m->getX());

        \Mockery::close();
    }

    public function testMock8 () {
        $m = \Mockery::mock(A::class);

        // 允许一个方法被模拟的次数
        $m->shouldReceive('getName')->once()->andReturn('a');
        $this->assertEquals('a', $m->getName());

        $this->expectException(\Mockery\Exception\InvalidCountException::class);
        $m->getName();

        \Mockery::close();
    }

    public function testMock9 () {
        $m = \Mockery::mock(A::class);

        // 允许一个方法被模拟的次数, 前两次返回 'a', 后两次返回 'b'
        $m->shouldReceive('getName')->times(2)->andReturn('a');
        $m->shouldReceive('getName')->times(1)->andReturn('b');
        $this->assertEquals('a', $m->getName());
        $this->assertEquals('a', $m->getName());
        $this->assertEquals('b', $m->getName());

        $this->expectException(\Mockery\Exception\InvalidCountException::class);
        $m->getName();

        \Mockery::close();
    }

    public function testMock10 () {
        $m = \Mockery::mock(C::class)->makePartial();
        $this->assertEquals(19, $m->proxyGetAge());

        // 模拟 protected 方法
        $m->shouldAllowMockingProtectedMethods();

        $m->allows()->getAge()->andReturn(18);
        $this->assertEquals(18, $m->proxyGetAge());

        \Mockery::close();
    }

    public function testMock11 () {
        $m = \Mockery::mock(C::class);

        // 先定义的先生效
        $m->shouldReceive('getName')->andReturn('duc');
        $m->shouldReceive('getName')->andReturn('aaa');

        $this->assertEquals('duc', $m->getName());

        \Mockery::close();
    }

    public function testMock12 () {
        $m = \Mockery::mock(C::class);

        // 设为默认，可被后定义的覆盖
        $m->shouldReceive('getName')->andReturn('duc')->byDefault();
        $m->shouldReceive('getName')->andReturn('aaa');

        $this->assertEquals('aaa', $m->getName());

        \Mockery::close();
    }

    public function testMock13 () {
        $m = \Mockery::spy(D::class);

        $this->assertNull($m->get());
        $this->assertNull($m->getAAAA());

        $m->shouldHaveReceived('get');
        $m->shouldHaveReceived('getAAAA');

        \Mockery::close();
    }

    public function testMock14 () {
        $m = \Mockery::spy(function () {
            return 'spy';
        });

        $this->assertEquals('spy', $m());

        $m->shouldHaveBeenCalled();

        \Mockery::close();
    }

    public function testMock15 () {
        $m = \Mockery::mock(E::class);

        // 允许这个方法走原来的定义
        $m->shouldReceive('getName')->once()->passthru();
        $this->assertEquals('duc', $m->getName());

        \Mockery::close();
    }

    public function testMock16 () {
        $m = \Mockery::mock(E::class);

        // 调用未定义的方法返回 Undefined 类
        $m->shouldIgnoreMissing()->asUndefined();

        $this->assertInstanceOf(Undefined::class, $m->getAge());

        \Mockery::close();
    }

    public function testMock17 () {
        $m = \Mockery::spy(E::class);

        $this->assertNull($m->get());

        $m->shouldHaveReceived('get')->once();

        \Mockery::close();
    }

    public function testMock18 () {
        $m = \Mockery::mock(E::class);

        // atLeast 至少被调用 1 次
        $m->shouldReceive('get')->atLeast()->once();
        $this->assertNull($m->get());
        $this->assertNull($m->get());

        \Mockery::close();
    }

    public function testMock19 () {
        $m = \Mockery::mock(E::class);

        // atLeast 至少被调用 1 次
        $m->shouldReceive('get')->atMost()->once();
        $this->assertNull($m->get());

        $this->expectException(\Mockery\Exception\InvalidCountException::class);
        $this->assertNull($m->get());

        \Mockery::close();
    }

    public function testMock20 () {
        $m = \Mockery::mock(E::class);

        $m->allows()->get()->once()->andReturnNull();
        $this->assertNull($m->get());

        $this->expectException(\Mockery\Exception\InvalidCountException::class);
        $this->assertNull($m->get());

        $m->mockery_verify();
    }

    public function testMock21 () {
        $m = \Mockery::mock(E::class);

        $m->shouldReceive(['get' => 1])->with('age');

        $this->assertEquals(1, $m->get('age'));

        $this->expectException(\Mockery\Exception\NoMatchingExpectationException::class);
        $this->assertEquals(1, $m->get('name'));

        \Mockery::close();
    }

    /**
     * @runInSeparateProcess
     * @preserveGlobalState disabled
     */
    public function testMock22 () {
        // 模拟静态方法，极不建议用静态调用。 static is bad！
        $m = \Mockery::mock('alias:'.F::class);
        $m->shouldReceive('get')->andReturn('get');

        $this->assertEquals('get', F::get());

        \Mockery::close();
    }

    public function testMock23 () {
        $m = \Mockery::mock(D::class);

        // between = atLeast()->times($minimum)->atMost()->times($maximum)
//        $m->allows()->get()->andReturn('duc')->between(1, 3);
        // or
        $m->shouldReceive(['get' => 'duc'])->between(1, 3);
        $this->assertEquals('duc', $m->get());
        $this->assertEquals('duc', $m->get());
        $this->assertEquals('duc', $m->get());

        \Mockery::close();
    }

    // 更多请查看源码
}

class A {
}

class B {
    public function getX () {
        return 'x';
    }

    public function getName () {
        return 'duc';
    }
}

class C {
    public function proxyGetAge () {
        return $this->getAge();
    }

    protected function getAge () {
        return 19;
    }
}

class D {
}

class E {
    public function getName () {
        return 'duc';
    }
}
```