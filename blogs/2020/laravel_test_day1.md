---
title: Laravel测试代码解读之还能这么玩Day1 
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - laravel
tags:
 - test
 - laravel
publish: true
---


### 读的是源码的 AuthAccessGateTest.php 文件

> policy 不一定要用户登录才能用，guests也能用policy，只要在对应的方法，允许用户为空 比如 before(?stdClass $user)


```php
[new AccessGateTestBeforeCallback, 'allowEverythingStatically'](null);
// $callable();

class AccessGateTestPolicyWithNonGuestBefore
{
    public function before(stdClass $user)
    {
        $_SERVER['__laravel.testBefore'] = true;
    }

  // 不需要用户登录
    public function edit(?stdClass $user, AccessGateTestDummy $dummy)
    {
        return true;
    }

  // 需要用户登录
    public function update($user, AccessGateTestDummy $dummy)
    {
        return true;
    }
}

// 是否需要用户
protected function parameterAllowsGuests($parameter)
{
  return ($parameter->getClass() && $parameter->allowsNull()) ||
    ($parameter->isDefaultValueAvailable() && is_null($parameter->getDefaultValue()));
}
```

Gate resources

```php
$gate->resource('test', AccessGateTestCustomResource::class, $abilities);
```

如果原来已经有result, gate 的 after 不会改变 结果， before 会

```php
protected function callAfterCallbacks($user, $ability, array $arguments, $result)
{
  foreach ($this->afterCallbacks as $after) {
    if (! $this->canBeCalledWithUser($user, $after)) {
      continue;
    }

    $afterResult = $after($user, $ability, $result, $arguments);

    $result = $result ?? $afterResult;
  }

  return $result;
}
```

```php
$gate->define('foo', AccessGateTestClass::class.'@foo');
```



// policy 可以用子类代替注册类

```php
public function getPolicyFor($class)
{
  if (is_object($class)) {
    $class = get_class($class);
  }

  if (! is_string($class)) {
    return;
  }

  if (isset($this->policies[$class])) {
    return $this->resolvePolicy($this->policies[$class]);
  }

  foreach ($this->guessPolicyName($class) as $guessedPolicy) {
    if (class_exists($guessedPolicy)) {
      return $this->resolvePolicy($guessedPolicy);
    }
  }

  foreach ($this->policies as $expected => $policy) {
    if (is_subclass_of($class, $expected)) {
      return $this->resolvePolicy($policy);
    }
  }
}

public function testPolicyClassesHandleChecksForInterfaces()
{
  $gate = $this->getBasicGate();

  // AccessGateTestSubDummy 是 AccessGateTestPolicy 子类
  $gate->policy(AccessGateTestDummyInterface::class, AccessGateTestPolicy::class);

  $this->assertTrue($gate->check('update', new AccessGateTestSubDummy));
}
```

Gate 的方法能被policy覆盖

```php
public function testPoliciesAlwaysOverrideClosuresWithSameName()
{
  $gate = $this->getBasicGate();

  $gate->define('update', function () {
    $this->fail();
  });

  $gate->policy(AccessGateTestDummy::class, AccessGateTestPolicy::class);

  $this->assertTrue($gate->check('update', new AccessGateTestDummy));
}

protected function resolveAuthCallback($user, $ability, array $arguments)
{
  // 如果 $policy 没有 update 方法就继续往下走
  if (isset($arguments[0]) &&
      ! is_null($policy = $this->getPolicyFor($arguments[0])) &&
      $callback = $this->resolvePolicyCallback($user, $ability, $arguments, $policy)) {
      return $callback;
  }

  if (isset($this->stringCallbacks[$ability])) {
    [$class, $method] = Str::parseCallback($this->stringCallbacks[$ability]);

    if ($this->canBeCalledWithUser($user, $class, $method ?: '__invoke')) {
      return $this->abilities[$ability];
    }
  }

  // 走到这里去执行 Gate 本身的 abilities 的update
  if (isset($this->abilities[$ability]) &&
      $this->canBeCalledWithUser($user, $this->abilities[$ability])) {
    return $this->abilities[$ability];
  }

  return function () {
    //
  };
}
```

forUser 设置了 $userResolver

```php
public function testForUserMethodAttachesANewUserToANewGateInstance()
{
    $gate = $this->getBasicGate();

    $gate->define('foo', function ($user) {
        $this->assertEquals(2, $user->id);

        return true;
    });

    $this->assertTrue($gate->forUser((object) ['id' => 2])->check('foo'));
}
```

详细的define定义方法

```php
public function testClassesCanBeDefinedAsCallbacksUsingAtNotationForGuests()
{
  $gate = new Gate(new Container, function () {
    //
  });

  $gate->define('foo', AccessGateTestClassForGuest::class.'@foo');
  $gate->define('obj_foo', [new AccessGateTestClassForGuest, 'foo']);
  $gate->define('static_foo', [AccessGateTestClassForGuest::class, 'staticFoo']);
  $gate->define('static_@foo', AccessGateTestClassForGuest::class.'@staticFoo');
  $gate->define('bar', AccessGateTestClassForGuest::class.'@bar');
  $gate->define('invokable', AccessGateTestGuestInvokableClass::class);
  $gate->define('nullable_invokable', AccessGateTestGuestNullableInvokable::class);
  $gate->define('absent_invokable', 'someAbsentClass');

  AccessGateTestClassForGuest::$calledMethod = '';

  $this->assertTrue($gate->check('foo'));
  $this->assertSame('foo was called', AccessGateTestClassForGuest::$calledMethod);

  $this->assertTrue($gate->check('static_foo'));
  $this->assertSame('static foo was invoked', AccessGateTestClassForGuest::$calledMethod);

  $this->assertTrue($gate->check('bar'));
  $this->assertSame('bar got invoked', AccessGateTestClassForGuest::$calledMethod);

  $this->assertTrue($gate->check('static_@foo'));
  $this->assertSame('static foo was invoked', AccessGateTestClassForGuest::$calledMethod);

  $this->assertTrue($gate->check('invokable'));
  $this->assertSame('__invoke was called', AccessGateTestGuestInvokableClass::$calledMethod);

  $this->assertTrue($gate->check('nullable_invokable'));
  $this->assertSame('Nullable __invoke was called', AccessGateTestGuestNullableInvokable::$calledMethod);

  $this->assertFalse($gate->check('absent_invokable'));
}
```


