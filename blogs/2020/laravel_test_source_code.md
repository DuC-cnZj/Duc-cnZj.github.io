---
title: Laravel测试代码解读之还能这么玩
date: '2020-06-28 12:14:19'
sidebar: false
categories:
 - 技术
tags:
 - laravel
 - test
 - 源码解读
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



## day2

```php
// m::any()
public function testDeleteExpiredMethodDeletesExpiredTokens()
{
  $repo = $this->getRepo();
  $repo->getConnection()->shouldReceive('table')->once()->with('table')->andReturn($query = m::mock(stdClass::class));
  $query->shouldReceive('where')->once()->with('created_at', '<', m::any())->andReturn($query);
  $query->shouldReceive('delete')->once();

  $repo->deleteExpired();
}
```



把 路由的参数直接传到 gate/policy

```php
public function testSimpleAbilityWithStringParameterFromRouteParameter()
{
    $this->gate()->define('view-dashboard', function ($user, $param) {
        return $param === 'true';
    });

    $this->router->get('dashboard/{route_parameter}', [
        'middleware' => Authorize::class.':view-dashboard,route_parameter',
        'uses' => function () {
            return 'success';
        },
    ]);

    $response = $this->router->dispatch(Request::create('dashboard/true', 'GET'));

    $this->assertEquals($response->content(), 'success');
}
```

```php

public function testModelUnauthorized()
{
  $this->expectException(AuthorizationException::class);
  $this->expectExceptionMessage('This action is unauthorized.');

  $post = new stdClass;

  $this->router->bind('post', function () use ($post) {
    return $post;
  });

  $this->gate()->define('edit', function ($user, $model) use ($post) {
    $this->assertSame($model, $post);

    return false;
  });

  $this->router->get('posts/{post}/edit', [
    'middleware' => [SubstituteBindings::class, Authorize::class.':edit,post'],
    'uses' => function () {
      return 'success';
    },
  ]);

  $this->router->dispatch(Request::create('posts/1/edit', 'GET'));
}

```

```php
// posts/{post}/edit posts/1/edit $value => 1
$this->router->bind('post', function ($value) use ($post) {
  return $post;
});
```



broadcaster 一个channel可以配置多个 guards。之后只要是guards的user都能拿到

```php
public function testRetrieveUserWithOneGuardUsingAStringForSpecifyingGuard()
{
  $this->broadcaster->channel('somechannel', function () {
    //
  }, ['guards' => 'myguard']);

  $request = m::mock(Request::class);
  $request->shouldReceive('user')
    ->once()
    ->with('myguard')
    ->andReturn(new DummyUser);

  $this->assertInstanceOf(
    DummyUser::class,
    $this->broadcaster->retrieveUser($request, 'somechannel')
  );
}
```



容器的 call

```php
public function testCallWithStaticMethodNameString()
{
  $container = new Container;
  $result = $container->call('Illuminate\Tests\Container\ContainerStaticMethodStub::inject');
  $this->assertInstanceOf(ContainerCallConcreteStub::class, $result[0]);
  $this->assertSame('taylor', $result[1]);
}
```





```php
public function testCallWithBoundMethod()
{
  $container = new Container;
  $container->bindMethod(ContainerTestCallStub::class.'@unresolvable', function ($stub) {
    return $stub->unresolvable('foo', 'bar');
  });
  $result = $container->call(ContainerTestCallStub::class.'@unresolvable');
  $this->assertEquals(['foo', 'bar'], $result);

  $container = new Container;
  $container->bindMethod(ContainerTestCallStub::class.'@unresolvable', function ($stub) {
    return $stub->unresolvable('foo', 'bar');
  });
  $result = $container->call([new ContainerTestCallStub, 'unresolvable']);
  $this->assertEquals(['foo', 'bar'], $result);

  $container = new Container;
  $result = $container->call([new ContainerTestCallStub, 'inject'], ['_stub' => 'foo', 'default' => 'bar']);
  $this->assertInstanceOf(ContainerCallConcreteStub::class, $result[0]);
  $this->assertSame('bar', $result[1]);

  $container = new Container;
  $result = $container->call([new ContainerTestCallStub, 'inject'], ['_stub' => 'foo']);
  $this->assertInstanceOf(ContainerCallConcreteStub::class, $result[0]);
  $this->assertSame('taylor', $result[1]);
}

class ContainerTestCallStub
{
    public function work()
    {
        return func_get_args();
    }

    public function inject(ContainerCallConcreteStub $stub, $default = 'taylor')
    {
        return func_get_args();
    }

    public function unresolvable($foo, $bar)
    {
        return func_get_args();
    }
}
```



容器已经实例化之后再 extend 会执行extend闭包

```php
public function testExtendInstanceRebindingCallback()
    {
        $_SERVER['_test_rebind'] = false;

        $container = new Container;
        $container->rebinding('foo', function () {
            $_SERVER['_test_rebind'] = true;
        });

        $obj = new stdClass;
        $container->instance('foo', $obj);

        $container->extend('foo', function ($obj, $container) {
            return $obj;
        });

        $this->assertTrue($_SERVER['_test_rebind']);
    }
```



容器的 rebound 会调用rebinding注册的方法，容器的 instance，bind，extend 都会触发 rebound

```php
$container->rebinding('foo', function () {
    $_SERVER['_test_rebind'] = true;
    dump(1);
});
```

```php
protected function rebound($abstract)
{
  $instance = $this->make($abstract);

  foreach ($this->getReboundCallbacks($abstract) as $callback) {
    call_user_func($callback, $this, $instance);
  }
}
```

容器tag

```php
public function testContainerTags()
{
  $container = new Container;
  $container->tag(ContainerImplementationTaggedStub::class, 'foo', 'bar');
  $container->tag(ContainerImplementationTaggedStubTwo::class, ['foo']);

  $this->assertCount(1, $container->tagged('bar'));
  $this->assertCount(2, $container->tagged('foo'));

  $fooResults = [];
  foreach ($container->tagged('foo') as $foo) {
    $fooResults[] = $foo;
  }

  $barResults = [];
  foreach ($container->tagged('bar') as $bar) {
    $barResults[] = $bar;
  }

  $this->assertInstanceOf(ContainerImplementationTaggedStub::class, $fooResults[0]);
  $this->assertInstanceOf(ContainerImplementationTaggedStub::class, $barResults[0]);
  $this->assertInstanceOf(ContainerImplementationTaggedStubTwo::class, $fooResults[1]);

  $container = new Container;
  $container->tag([ContainerImplementationTaggedStub::class, ContainerImplementationTaggedStubTwo::class], ['foo']);
  $this->assertCount(2, $container->tagged('foo'));

  $fooResults = [];
  // 用tagged批量实例化
  foreach ($container->tagged('foo') as $foo) {
    $fooResults[] = $foo;
  }

  $this->assertInstanceOf(ContainerImplementationTaggedStub::class, $fooResults[0]);
  $this->assertInstanceOf(ContainerImplementationTaggedStubTwo::class, $fooResults[1]);

  $this->assertCount(0, $container->tagged('this_tag_does_not_exist'));
}
```

bindIf 如果绑定了就不会再绑定了

```php
public function testBindIfDoesntRegisterIfServiceAlreadyRegistered()
{
  $container = new Container;
  $container->bind('name', function () {
    return 'Taylor';
  });
  $container->bindIf('name', function () {
    return 'Dayle';
  });

  $this->assertSame('Taylor', $container->make('name'));
}
```

如果单例make的时候传了第二个参数或者有上下文绑定，那么这个单例是不会被加入到instances数组里面的，下次获取的时候又会实例化

```php
if ($this->isShared($abstract) && ! $needsContextualBuild) {
    $this->instances[$abstract] = $object;
}

public function test()
{
  $container = new Container;
  $container->singleton('class', function () {
    return new stdClass;
  });
  $firstInstantiation = $container->make('class', ['a']);
  $secondInstantiation = $container->make('class');
  $this->assertNotSame($firstInstantiation, $secondInstantiation);
}
```

如果在rebinding之前bind过那么会make实例

```php
public function testReboundListeners()
{
  unset($_SERVER['__test.rebind']);

  $container = new Container;
  $container->bind('foo', function () {
    //
  });
  $container->rebinding('foo', function () {
    $_SERVER['__test.rebind'] = true;
  });
  $container->bind('foo', function () {
    //
  });

  $this->assertTrue($_SERVER['__test.rebind']);
}
```



容器绑定 bind 如果传class ，那么make的时候参数必须以 key=>value 传，如果是闭包，则不用,闭包不能传第三个参数 `    $container->bind('foo', function ($app, $config, $a) {}` $a 解析不到的

```php
// 闭包
if ($concrete instanceof Closure) {
  return $concrete($this, $this->getLastParameterOverride());
}

```



```php
public function testResolvingWithUsingAnInterface()
{
    $container = new Container;
    $container->bind(IContainerContractStub::class, ContainerInjectVariableStubWithInterfaceImplementation::class);
    $instance = $container->make(IContainerContractStub::class, ['something' => 'laurence']);
    $this->assertSame('laurence', $instance->something);
}

class ContainerInjectVariableStubWithInterfaceImplementation implements IContainerContractStub
{
    public $something;

    public function __construct(ContainerConcreteStub $concrete, $something)
    {
        $this->something = $something;
    }
}

public function testNestedParameterOverride()
{
    $container = new Container;
    $container->bind('foo', function ($app, $config) {
        return $app->make('bar', ['name' => 'Taylor']);
    });
    $container->bind('bar', function ($app, $config) {
        return $config;
    });

    $this->assertEquals(['name' => 'Taylor'], $container->make('foo', ['something']));
}
```



当使用了 when give的时候  $container->instance无效, 只要 有when give 那么give里面的都会重新实例化，不管之前有没有实例化过

```php
public function testContextualBindingWorksOnExistingAliasedInstances()
{
  $container = new Container;

  $container->instance('stub', new ContainerImplementationStub);
  $container->alias('stub', IContainerContextContractStub::class);

  $container->when(ContainerTestContextInjectOne::class)->needs(IContainerContextContractStub::class)->give(ContainerContextImplementationStubTwo::class);

  $this->assertInstanceOf(
    ContainerContextImplementationStubTwo::class,
    $container->make(ContainerTestContextInjectOne::class)->impl
  );
}
```

```php
public function testContextualBindingWorksForNewlyInstancedBindings()
{
  $container = new Container;

  $container->when(ContainerTestContextInjectOne::class)->needs(IContainerContextContractStub::class)->give(ContainerContextImplementationStubTwo::class);

  $container->instance(IContainerContextContractStub::class, new ContainerImplementationStub);

  $this->assertInstanceOf(
    ContainerContextImplementationStubTwo::class,
    $container->make(ContainerTestContextInjectOne::class)->impl
  );
}
```

细品 when->give 难

> 调试下就懂了
>
> ​    $container->make(ContainerTestContextInjectOne::class)
>
> ​    $container->make(ContainerTestContextInjectTwo::class)

```php
public function testContextualBindingWorksOnNewAliasedBindings()
{
  $container = new Container;

  $container->when(ContainerTestContextInjectOne::class)->needs(IContainerContextContractStub::class)->give(ContainerContextImplementationStubTwo::class);

  $container->bind('stub', ContainerContextImplementationStub::class);
  $container->alias('stub', IContainerContextContractStub::class);

  $this->assertInstanceOf(
    ContainerContextImplementationStubTwo::class,
    $container->make(ContainerTestContextInjectOne::class)->impl
  );
}
```

When->give 注入普通变量

```php
public function testContainerCanInjectSimpleVariable()
{
  $container = new Container;
  $container->when(ContainerInjectVariableStub::class)->needs('$something')->give(100);
  $instance = $container->make(ContainerInjectVariableStub::class);
  $this->assertEquals(100, $instance->something);

  $container = new Container;
  $container->when(ContainerInjectVariableStub::class)->needs('$something')->give(function ($container) {
    return $container->make(ContainerConcreteStub::class);
  });
  $instance = $container->make(ContainerInjectVariableStub::class);
  $this->assertInstanceOf(ContainerConcreteStub::class, $instance->something);
}
```





sqlite 也可以设置外键约束

```php
public function testSqliteForeignKeyConstraints()
{
  $this->db->addConnection([
    'url' => 'sqlite:///:memory:?foreign_key_constraints=true',
  ], 'constraints_set');

  $this->assertEquals(0, $this->db->getConnection()->select('PRAGMA foreign_keys')[0]->foreign_keys);

  $this->assertEquals(1, $this->db->getConnection('constraints_set')->select('PRAGMA foreign_keys')[0]->foreign_keys);
}
```





DB -> pretend (dry run mode)

```php
$queries = $connection->pretend(function ($connection) {
  $connection->select('foo bar', ['baz']);
});
```



withDefault 可以塞数组可以塞闭包 文档有

```php
public function testBelongsToWithArrayDefault()
{
  $relation = $this->getRelation()->withDefault(['username' => 'taylor']);

  $this->builder->shouldReceive('first')->once()->andReturnNull();

  $newModel = new EloquentBelongsToModelStub;

  $this->related->shouldReceive('newInstance')->once()->andReturn($newModel);

  $this->assertSame($newModel, $relation->getResults());

  $this->assertSame('taylor', $newModel->username);
}
```



findOrFail() 传数组的时候会匹配数量，如果数量不一致会ModelNotFoundException

> [1,2] 最后只有1个结果[1] 就会报错

```php
public function testFindOrFailMethodWithManyUsingCollectionThrowsModelNotFoundException()
{
  $this->expectException(ModelNotFoundException::class);

  $builder = m::mock(Builder::class.'[get]', [$this->getMockQueryBuilder()]);
  $builder->setModel($this->getMockModel());
  $builder->getQuery()->shouldReceive('whereIn')->once()->with('foo_table.foo', [1, 2]);
  $builder->shouldReceive('get')->with(['column'])->andReturn(new Collection([1]));
  $builder->findOrFail(new Collection([1, 2]), ['column']);
}
```



chunk 内容return false 直接中断

```php
public function testChunkCanBeStoppedByReturningFalse()
{
  $builder = m::mock(Builder::class.'[forPage,get]', [$this->getMockQueryBuilder()]);
  $builder->getQuery()->orders[] = ['column' => 'foobar', 'direction' => 'asc'];

  $chunk1 = new Collection(['foo1', 'foo2']);
  $chunk2 = new Collection(['foo3']);
  $builder->shouldReceive('forPage')->once()->with(1, 2)->andReturnSelf();
  $builder->shouldReceive('forPage')->never()->with(2, 2);
  $builder->shouldReceive('get')->times(1)->andReturn($chunk1);

  $callbackAssertor = m::mock(stdClass::class);
  $callbackAssertor->shouldReceive('doSomething')->once()->with($chunk1);
  $callbackAssertor->shouldReceive('doSomething')->never()->with($chunk2);

  $builder->chunk(2, function ($results) use ($callbackAssertor) {
    $callbackAssertor->doSomething($results);

    return false;
  });
}
```



chunkById 第二个参数是'id'字段的名称

```php
public function testChunkPaginatesUsingIdWithLastChunkComplete()
{
  $builder = m::mock(Builder::class.'[forPageAfterId,get]', [$this->getMockQueryBuilder()]);
  $builder->getQuery()->orders[] = ['column' => 'foobar', 'direction' => 'asc'];

  $chunk1 = new Collection([(object) ['someIdField' => 1], (object) ['someIdField' => 2]]);
  $chunk2 = new Collection([(object) ['someIdField' => 10], (object) ['someIdField' => 11]]);
  $chunk3 = new Collection([]);
  $builder->shouldReceive('forPageAfterId')->once()->with(2, 0, 'someIdField')->andReturnSelf();
  $builder->shouldReceive('forPageAfterId')->once()->with(2, 2, 'someIdField')->andReturnSelf();
  $builder->shouldReceive('forPageAfterId')->once()->with(2, 11, 'someIdField')->andReturnSelf();
  $builder->shouldReceive('get')->times(3)->andReturn($chunk1, $chunk2, $chunk3);

  $callbackAssertor = m::mock(stdClass::class);
  $callbackAssertor->shouldReceive('doSomething')->once()->with($chunk1);
  $callbackAssertor->shouldReceive('doSomething')->once()->with($chunk2);
  $callbackAssertor->shouldReceive('doSomething')->never()->with($chunk3);

  $builder->chunkById(2, function ($results) use ($callbackAssertor) {
    $callbackAssertor->doSomething($results);
  }, 'someIdField');
}
```



orWhere 可以高阶

```php
public function testRealQueryHigherOrderOrWhereScopes()
{
  $model = new EloquentBuilderTestHigherOrderWhereScopeStub;
  $this->mockConnectionForModel($model, 'SQLite');
  $query = $model->newQuery()->one()->orWhere->two();
  $this->assertSame('select * from "table" where "one" = ? or ("two" = ?)', $query->toSql());
}

public function testRealQueryChainedHigherOrderOrWhereScopes()
{
  $model = new EloquentBuilderTestHigherOrderWhereScopeStub;
  $this->mockConnectionForModel($model, 'SQLite');
  $query = $model->newQuery()->one()->orWhere->two()->orWhere->three();
  $this->assertSame('select * from "table" where "one" = ? or ("two" = ?) or ("three" = ?)', $query->toSql());
}
```



`Model::get()`返回的是`Illuminate\Database\Eloquent\Collection`





hasManyThrough 如果中间数据被软删了，还是能拿到第三层的数据的

```php
public function testIntermediateSoftDeletesAreIgnored()
{
    $this->seedData();
    HasManyThroughSoftDeletesTestUser::first()->delete();

    $posts = HasManyThroughSoftDeletesTestCountry::first()->posts;

    $this->assertSame('A title', $posts[0]->title);
    $this->assertCount(2, $posts);
}
```





isDirty 是比较 cast 或者 date 之后的值

```php
public function testDirtyOnCastOrDateAttributes()
{
  $model = new EloquentModelCastingStub;
  $model->setDateFormat('Y-m-d H:i:s');
  $model->boolAttribute = 1;
  $model->foo = 1;
  $model->bar = '2017-03-18';
  $model->dateAttribute = '2017-03-18';
  $model->datetimeAttribute = '2017-03-23 22:17:00';
  $model->syncOriginal();

  $model->boolAttribute = true;
  $model->foo = true;
  $model->bar = '2017-03-18 00:00:00';
  $model->dateAttribute = '2017-03-18 00:00:00';
  $model->datetimeAttribute = null;

  $this->assertTrue($model->isDirty());
  $this->assertTrue($model->isDirty('foo'));
  $this->assertTrue($model->isDirty('bar'));
  $this->assertFalse($model->isDirty('boolAttribute'));
  $this->assertFalse($model->isDirty('dateAttribute'));
  $this->assertTrue($model->isDirty('datetimeAttribute'));
}
```

model destroy 返回删除的数量





更新会去判断dirty属性并且更新dirty属性内容，如果调用 syncOriginal 会出现脏值，不会被更新到库

```php
public function testUpdateProcess()
{
  /** @var Model $model */
  $model = $this->getMockBuilder(EloquentModelStub::class)->setMethods(['newModelQuery', 'updateTimestamps'])->getMock();
  $query = m::mock(Builder::class);
  $query->shouldReceive('where')->once()->with('id', '=', 1);
  $query->shouldReceive('update')->once()->with(['name' => 'taylor'])->andReturn(1);
  $model->expects($this->once())->method('newModelQuery')->willReturn($query);
  $model->expects($this->once())->method('updateTimestamps');
  $model->setEventDispatcher($events = m::mock(Dispatcher::class));
  $events->shouldReceive('until')->once()->with('eloquent.saving: '.get_class($model), $model)->andReturn(true);
  $events->shouldReceive('until')->once()->with('eloquent.updating: '.get_class($model), $model)->andReturn(true);
  $events->shouldReceive('dispatch')->once()->with('eloquent.updated: '.get_class($model), $model)->andReturn(true);
  $events->shouldReceive('dispatch')->once()->with('eloquent.saved: '.get_class($model), $model)->andReturn(true);

  $model->id = 1;
  $model->foo = 'bar';
  // make sure foo isn't synced so we can test that dirty attributes only are updated
  $model->syncOriginal();
  $model->name = 'taylor';
  $model->exists = true;
  $this->assertTrue($model->save());
}
```



如果手动传入  created_at updated_at 时间戳 是不会被框架覆盖的，即便是错的数据

```php
public function testUpdateProcessDoesntOverrideTimestamps()
{
  $model = $this->getMockBuilder(EloquentModelStub::class)->setMethods(['newModelQuery'])->getMock();
  $query = m::mock(Builder::class);
  $query->shouldReceive('where')->once()->with('id', '=', 1);
  $query->shouldReceive('update')->once()->with(['created_at' => 'foo', 'updated_at' => 'bar'])->andReturn(1);
  $model->expects($this->once())->method('newModelQuery')->willReturn($query);
  $model->setEventDispatcher($events = m::mock(Dispatcher::class));
  $events->shouldReceive('until');
  $events->shouldReceive('dispatch');

  $model->id = 1;
  $model->syncOriginal();
  $model->created_at = 'foo';
  $model->updated_at = 'bar';
  $model->exists = true;
  $this->assertTrue($model->save());
}
```



Save 不存在的数据会调用 insertGetId 会更新 model 的id

```php
public function testPushNoRelations()
{
    /** @var Model $model */
    $model = $this->getMockBuilder(EloquentModelStub::class)->setMethods(['newModelQuery', 'updateTimestamps', 'refresh'])->getMock();
    $query = m::mock(Builder::class);
    $query->shouldReceive('insertGetId')->once()->with(['name' => 'taylor'], 'id')->andReturn(1);
    $query->shouldReceive('getConnection')->once();
    $model->expects($this->once())->method('newModelQuery')->willReturn($query);
    $model->expects($this->once())->method('updateTimestamps');

    $model->name = 'taylor';
    $model->exists = false;

    $this->assertTrue($model->push());
    $this->assertEquals(1, $model->id);
    $this->assertTrue($model->exists);
}
```



push 能更新关联关系，如果关联不存在也能创建

```php
public function testPushOneRelation()
{
  /** @var Model $model */
  $related1 = $this->getMockBuilder(EloquentModelStub::class)->setMethods(['newModelQuery', 'updateTimestamps', 'refresh'])->getMock();
  $query = m::mock(Builder::class);
  $query->shouldReceive('insertGetId')->once()->with(['name' => 'related1'], 'id')->andReturn(2);
  $query->shouldReceive('getConnection')->once();
  $related1->expects($this->once())->method('newModelQuery')->willReturn($query);
  $related1->expects($this->once())->method('updateTimestamps');
  $related1->name = 'related1';
  $related1->exists = false;

  $model = $this->getMockBuilder(EloquentModelStub::class)->setMethods(['newModelQuery', 'updateTimestamps', 'refresh'])->getMock();
  $query = m::mock(Builder::class);
  $query->shouldReceive('insertGetId')->once()->with(['name' => 'taylor'], 'id')->andReturn(1);
  $query->shouldReceive('getConnection')->once();
  $model->expects($this->once())->method('newModelQuery')->willReturn($query);
  $model->expects($this->once())->method('updateTimestamps');

  $model->name = 'taylor';
  $model->exists = false;
  $model->setRelation('relationOne', $related1);

  $this->assertTrue($model->push());
  $this->assertEquals(1, $model->id);
  $this->assertTrue($model->exists);
  $this->assertEquals(2, $model->relationOne->id);
  $this->assertTrue($model->relationOne->exists);
  $this->assertEquals(2, $related1->id);
  $this->assertTrue($related1->exists);
}
```



push 能设置不存在的关联并且更新他们

```php
public function testPushEmptyManyRelation()
{
  /** @var Model $model */
  $model = $this->getMockBuilder(EloquentModelStub::class)->setMethods(['newModelQuery', 'updateTimestamps', 'refresh'])->getMock();
  $query = m::mock(Builder::class);
  $query->shouldReceive('insertGetId')->once()->with(['name' => 'taylor'], 'id')->andReturn(1);
  $query->shouldReceive('getConnection')->once();
  $model->expects($this->once())->method('newModelQuery')->willReturn($query);
  $model->expects($this->once())->method('updateTimestamps');

  $model->name = 'taylor';
  $model->exists = false;
  $model->setRelation('relationMany', new Collection([]));

  $this->assertTrue($model->push());
  $this->assertEquals(1, $model->id);
  $this->assertTrue($model->exists);
  $this->assertCount(0, $model->relationMany);
}
```



model setHidden能隐藏关联属性

```php
public function testHiddenCanAlsoExcludeRelationships()
{
  $model = new EloquentModelStub;
  $model->name = 'Taylor';
  $model->setRelation('foo', ['bar']);
  $model->setHidden(['foo', 'list_items', 'password']);
  $array = $model->toArray();

  $this->assertEquals(['name' => 'Taylor'], $array);
}
```



filter sql 冲突 可以加这个

```php
public function testQualifyColumn()
{
  $model = new EloquentModelStub;

  $this->assertSame('stub.column', $model->qualifyColumn('column'));
}
```



model 可以 用 -> 设置属性，底层用了 `Arr::set`

```php
public function testFillingJSONAttributes()
{
  $model = new EloquentModelStub;
  $model->fillable(['meta->name', 'meta->price', 'meta->size->width']);
  $model->fill(['meta->name' => 'foo', 'meta->price' => 'bar', 'meta->size->width' => 'baz']);
  $this->assertEquals(
    ['meta' => json_encode(['name' => 'foo', 'price' => 'bar', 'size' => ['width' => 'baz']])],
    $model->toArray()
  );

  $model = new EloquentModelStub(['meta' => json_encode(['name' => 'Taylor'])]);
  $model->fillable(['meta->name', 'meta->price', 'meta->size->width']);
  $model->fill(['meta->name' => 'foo', 'meta->price' => 'bar', 'meta->size->width' => 'baz']);
  $this->assertEquals(
    ['meta' => json_encode(['name' => 'foo', 'price' => 'bar', 'size' => ['width' => 'baz']])],
    $model->toArray()
  );
}



// test
{
    /**
     * A basic test example.
     *
     * @return void
     */
    public function testInspiringCommand()
    {
        $m = (new MyModel());
        $m->fill([
            'info->name' => 'duc',
        ]);
    }
}

class MyModel extends Model {
    protected $guarded = [];

    protected $casts = [
//      'info' => 'array'
    ];
}

```



**下划线开头**的参数都会被过滤掉

```php
public function testUnderscorePropertiesAreNotFilled()
{
    $model = new EloquentModelStub;
    $model->fill(['_method' => 'PUT']);
    $this->assertEquals([], $model->getAttributes());
}
```



同一个字段同时设置了 fillable 和 guarded ， fillable 优先级高

```php
public function testFillableOverridesGuarded()
{
  $model = new EloquentModelStub;
  $model->guard(['name', 'age']);
  $model->fillable(['age', 'foo']);
  $model->fill(['name' => 'foo', 'age' => 'bar', 'foo' => 'bar']);
  $this->assertFalse(isset($model->name));
  $this->assertSame('bar', $model->age);
  $this->assertSame('bar', $model->foo);
}
```

unguarded 知识点

```php
public function testUnguardedRunsCallbackWhileBeingUnguarded()
{
  $model = Model::unguarded(function () {
    return (new EloquentModelStub)->guard(['*'])->fill(['name' => 'Taylor']);
  });
  $this->assertSame('Taylor', $model->name);
  $this->assertFalse(Model::isUnguarded());
}
```





Event 推送到队列

```php
public function testQueuedEventsAreFired()
{
  unset($_SERVER['__event.test']);
  $d = new Dispatcher;
  $d->listen('update', function ($name) {
    $_SERVER['__event.test'] = $name;
  });
  $d->push('update', ['name' => 'taylor']);
  $d->listen('update', function ($name) {
    $_SERVER['__event.test'] .= '_'.$name;
  });

  $this->assertFalse(isset($_SERVER['__event.test']));
  $d->flush('update');
  $d->listen('update', function ($name) {
    $_SERVER['__event.test'] .= $name;
  });
  $this->assertSame('taylor_taylor', $_SERVER['__event.test']);
}
```



Wildcard event 和 普通 event 区别：传参

```php
public function testEventPassedFirstToWildcards()
{
  $d = new Dispatcher;
  $d->listen('foo.*', function ($event, $data) {
    $this->assertSame('foo.bar', $event);
    $this->assertEquals(['first', 'second'], $data);
  });
  $d->dispatch('foo.bar', ['first', 'second']);

  $d = new Dispatcher;
  $d->listen('foo.bar', function ($first, $second) {
    $this->assertSame('first', $first);
    $this->assertSame('second', $second);
  });
  $d->dispatch('foo.bar', ['first', 'second']);
}
```



如果是用 obj的方式触发的event，第二个参数会被忽略

```php
public function testEventClassesArePayload()
{
  unset($_SERVER['__event.test']);
  $d = new Dispatcher;
  $d->listen(ExampleEvent::class, function ($payload) {
    $_SERVER['__event.test'] = $payload;
  });
  $d->dispatch($e = new ExampleEvent, ['foo']);

  $this->assertSame($e, $_SERVER['__event.test']);
}
```

学到了！！ Event 可以监听 interface

```php
public function testInterfacesWork()
{
  unset($_SERVER['__event.test']);
  $d = new Dispatcher;
  $d->listen(SomeEventInterface::class, function () {
    $_SERVER['__event.test'] = 'bar';
  });
  $d->dispatch(new AnotherEvent);

  $this->assertSame('bar', $_SERVER['__event.test']);
}
```

phpunit 断言输出字符串

```php
public function testCanFailSilent()
{
  $this->expectOutputString('da');

	echo "da";
}
```

env 判断还能正则

```php
public function testEnvironment()
{
  $app = new Application;
  $app['env'] = 'foo';

  $this->assertSame('foo', $app->environment());

  $this->assertTrue($app->environment('foo'));
  $this->assertTrue($app->environment('f*'));
  $this->assertTrue($app->environment('foo', 'bar'));
  $this->assertTrue($app->environment(['foo', 'bar']));

  $this->assertFalse($app->environment('qux'));
  $this->assertFalse($app->environment('q*'));
  $this->assertFalse($app->environment('qux', 'bar'));
  $this->assertFalse($app->environment(['qux', 'bar']));
}
```



系统路径都可以通过env设置

```php
public function testEnvPathsAreUsedForCachePathsWhenSpecified()
{
  $app = new Application('/base/path');
  $_SERVER['APP_SERVICES_CACHE'] = '/absolute/path/services.php';
  $_SERVER['APP_PACKAGES_CACHE'] = '/absolute/path/packages.php';
  $_SERVER['APP_CONFIG_CACHE'] = '/absolute/path/config.php';
  $_SERVER['APP_ROUTES_CACHE'] = '/absolute/path/routes.php';
  $_SERVER['APP_EVENTS_CACHE'] = '/absolute/path/events.php';

  $this->assertSame('/absolute/path/services.php', $app->getCachedServicesPath());
  $this->assertSame('/absolute/path/packages.php', $app->getCachedPackagesPath());
  $this->assertSame('/absolute/path/config.php', $app->getCachedConfigPath());
  $this->assertSame('/absolute/path/routes.php', $app->getCachedRoutesPath());
  $this->assertSame('/absolute/path/events.php', $app->getCachedEventsPath());

  unset(
    $_SERVER['APP_SERVICES_CACHE'],
    $_SERVER['APP_PACKAGES_CACHE'],
    $_SERVER['APP_CONFIG_CACHE'],
    $_SERVER['APP_ROUTES_CACHE'],
    $_SERVER['APP_EVENTS_CACHE']
  );
}
```

使用表单验证可以直接通过 `$request->validated();` 获取结果

```php
public function __invoke (TestRequest $request) {
    $result = $request->validated();
}
```

可以在 formRequest中替换input内容

注意 `$request->replace($replace);`会替换调原先的所有input内容

```php
public function passedValidation()
{
  $this->replace(['name' => 'Adam']);
}
```

自带的cache中间件处理

```php
Illuminate\Http\Middleware\SetCacheHeaders
  
public function testAddHeaders()
{
  $response = (new Cache)->handle(new Request, function () {
    return new Response('some content');
  }, 'max_age=100;s_maxage=200;etag=ABC');

  $this->assertSame('"ABC"', $response->getEtag());
  $this->assertSame('max-age=100, public, s-maxage=200', $response->headers->get('Cache-Control'));
}
```



Basic 认证默认使用 email password 校验

```php
public function testBasicAuthRespectsAdditionalConditions()
{
  AuthenticationTestUser::create([
    'username' => 'username2',
    'email' => 'email2',
    'password' => bcrypt('password'),
    'is_active' => false,
  ]);

  $this->get('basicWithCondition', [
    'Authorization' => 'Basic '.base64_encode('email2:password2'),
  ])->assertStatus(401);

  $this->get('basicWithCondition', [
    'Authorization' => 'Basic '.base64_encode('email:password'),
  ])->assertStatus(200);
}
```



### Resource

可以直接根据 resource 生成 url

```php
public function testResourceIsUrlRoutable()
{
  $post = new PostResource(new Post([
    'id' => 5,
    'title' => 'Test Title',
  ]));

  $this->assertSame('http://localhost/post/5', url('/post', $post));
}


public function testNamedRoutesAreUrlRoutable()
{
  $post = new PostResource(new Post([
    'id' => 5,
    'title' => 'Test Title',
  ]));

  Route::get('/post/{id}', function () use ($post) {
    return route('post.show', $post);
  })->name('post.show');

  $response = $this->withoutExceptionHandling()->get('/post/1');

  $this->assertSame('http://localhost/post/5', $response->original);
}
```



```php

public function testCustomHeadersMayBeSetOnResponses()
{
  Route::get('/', function () {
    return (new PostResource(new Post([
      'id' => 5,
      'title' => 'Test Title',
    ])))->response()->setStatusCode(202)->header('X-Custom', 'True');
  });

  $response = $this->withoutExceptionHandling()->get(
    '/', ['Accept' => 'application/json']
  );

  $response->assertStatus(202);
  $response->assertHeader('X-Custom', 'True');
}
```

返回内容保留 query 信息

```php
public function testPaginatorResourceCanPreserveQueryParameters()
{
  Route::get('/', function () {
    $collection = collect([new Post(['id' => 2, 'title' => 'Laravel Nova'])]);
    $paginator = new LengthAwarePaginator(
      $collection, 3, 1, 2
    );

    return PostCollectionResource::make($paginator)->preserveQuery();
  });

  $response = $this->withoutExceptionHandling()->get(
    '/?framework=laravel&author=Otwell&page=2', ['Accept' => 'application/json']
  );

  $response->assertStatus(200);

  $response->assertJson([
    'data' => [
      [
        'id' => 2,
        'title' => 'Laravel Nova',
      ],
    ],
    'links' => [
      'first' => '/?framework=laravel&author=Otwell&page=1',
      'last' => '/?framework=laravel&author=Otwell&page=3',
      'prev' => '/?framework=laravel&author=Otwell&page=1',
      'next' => '/?framework=laravel&author=Otwell&page=3',
    ],
    'meta' => [
      'current_page' => 2,
      'from' => 2,
      'last_page' => 3,
      'path' => '/',
      'per_page' => 1,
      'to' => 2,
      'total' => 3,
    ],
  ]);
}
```

```php
public function testPaginatorResourceCanReceiveQueryParameters()
{
  Route::get('/', function () {
    $collection = collect([new Post(['id' => 2, 'title' => 'Laravel Nova'])]);
    $paginator = new LengthAwarePaginator(
      $collection, 3, 1, 2
    );

    return PostCollectionResource::make($paginator)->withQuery(['author' => 'Taylor']);
  });

  $response = $this->withoutExceptionHandling()->get(
    '/?framework=laravel&author=Otwell&page=2', ['Accept' => 'application/json']
  );

  $response->assertStatus(200);

  $response->assertJson([
    'data' => [
      [
        'id' => 2,
        'title' => 'Laravel Nova',
      ],
    ],
    'links' => [
      'first' => '/?author=Taylor&page=1',
      'last' => '/?author=Taylor&page=3',
      'prev' => '/?author=Taylor&page=1',
      'next' => '/?author=Taylor&page=3',
    ],
    'meta' => [
      'current_page' => 2,
      'from' => 2,
      'last_page' => 3,
      'path' => '/',
      'per_page' => 1,
      'to' => 2,
      'total' => 3,
    ],
  ]);
}
```

 `protected $preserveKeys = true;` 可以保留key

```php
class ResourceWithPreservedKeys extends PostResource
{
    protected $preserveKeys = true;

    public function toArray($request)
    {
        return $this->resource;
    }
}

public function testKeysArePreservedInAnAnonymousColletionIfTheResourceIsFlaggedToPreserveKeys()
{
  $data = Collection::make([
    [
      'id' => 1,
      'authorId' => 5,
      'bookId' => 22,
    ],
    [
      'id' => 2,
      'authorId' => 5,
      'bookId' => 15,
    ],
    [
      'id' => 3,
      'authorId' => 42,
      'bookId' => 12,
    ],
  ])->keyBy->id;

  Route::get('/', function () use ($data) {
    return ResourceWithPreservedKeys::collection($data);
  });

  $response = $this->withoutExceptionHandling()->get(
    '/', ['Accept' => 'application/json']
  );

  $response->assertStatus(200);

  $response->assertJson(['data' => $data->toArray()]);
}
```

个不同国家的人用不同的语言发邮件

```php
class TestEmailLocaleUser extends Model implements HasLocalePreference
{
    protected $fillable = [
        'email',
        'email_locale',
    ];

    public function preferredLocale()
    {
        return $this->email_locale;
    }
}

public function testLocaleIsSentWithModelPreferredLocale()
{
  $recipient = new TestEmailLocaleUser([
    'email' => 'test@mail.com',
    'email_locale' => 'ar',
  ]);

  Mail::to($recipient)->send(new TestMail);

  $this->assertStringContainsString('esm',
                                    app('mailer')->getSwiftMailer()->getTransport()->messages()[0]->getBody()
                                   );
}
```

执行 job  时，如果之前序列化的model不存在了，则$deleteWhenMissingModels可以删除该job

```php
class CallQueuedHandlerExceptionThrower
{
    public $deleteWhenMissingModels = true;

    public function handle()
    {
        //
    }

    public function __wakeup()
    {
        throw new ModelNotFoundException('Foo');
    }
}
```



链式队列 `job chain` 多种写法

> 如果第一个job失败了，那么后面的就不会继续了

```php
public function testJobsCanBeChainedOnSuccessUsingPendingChain()
{
  JobChainingTestFirstJob::withChain([
    new JobChainingTestSecondJob,
  ])->dispatch();

  $this->assertTrue(JobChainingTestFirstJob::$ran);
  $this->assertTrue(JobChainingTestSecondJob::$ran);
}

public function testJobsCanBeChainedOnSuccessUsingBusFacade()
{
  Bus::dispatchChain([
    new JobChainingTestFirstJob(),
    new JobChainingTestSecondJob(),
  ]);

  $this->assertTrue(JobChainingTestFirstJob::$ran);
  $this->assertTrue(JobChainingTestSecondJob::$ran);
}

public function testJobsCanBeChainedOnSuccessWithSeveralJobs()
{
  JobChainingTestFirstJob::dispatch()->chain([
    new JobChainingTestSecondJob,
    new JobChainingTestThirdJob,
  ]);

  $this->assertTrue(JobChainingTestFirstJob::$ran);
  $this->assertTrue(JobChainingTestSecondJob::$ran);
  $this->assertTrue(JobChainingTestThirdJob::$ran);
}

public function testJobsCanBeChainedOnSuccessUsingHelper()
{
  dispatch(new JobChainingTestFirstJob)->chain([
    new JobChainingTestSecondJob,
  ]);

  $this->assertTrue(JobChainingTestFirstJob::$ran);
  $this->assertTrue(JobChainingTestSecondJob::$ran);
}

public function testJobsCanBeChainedViaQueue()
{
  Queue::connection('sync')->push((new JobChainingTestFirstJob)->chain([
    new JobChainingTestSecondJob,
  ]));

  $this->assertTrue(JobChainingTestFirstJob::$ran);
  $this->assertTrue(JobChainingTestSecondJob::$ran);
}
```

如果第一个job被释放了，那么第二个不会执行

```php
public function testSecondJobIsNotFiredIfFirstReleased()
{
  Queue::connection('sync')->push((new JobChainingTestReleasingJob)->chain([
    new JobChainingTestSecondJob,
  ]));

  $this->assertFalse(JobChainingTestSecondJob::$ran);
}
```

3个 job 如果第一个成功第二个失败，那么第三个不执行

```php
public function testThirdJobIsNotFiredIfSecondFails()
{
  Queue::connection('sync')->push((new JobChainingTestFirstJob)->chain([
    new JobChainingTestFailingJob,
    new JobChainingTestThirdJob,
  ]));

  $this->assertTrue(JobChainingTestFirstJob::$ran);
  $this->assertFalse(JobChainingTestThirdJob::$ran);
}
```

多个job使用相同队列、使用不同配置

```php

public function testChainJobsUseSameConfig()
{
  JobChainingTestFirstJob::dispatch()->allOnQueue('some_queue')->allOnConnection('sync1')->chain([
    new JobChainingTestSecondJob,
    new JobChainingTestThirdJob,
  ]);

  $this->assertSame('some_queue', JobChainingTestFirstJob::$usedQueue);
  $this->assertSame('sync1', JobChainingTestFirstJob::$usedConnection);

  $this->assertSame('some_queue', JobChainingTestSecondJob::$usedQueue);
  $this->assertSame('sync1', JobChainingTestSecondJob::$usedConnection);

  $this->assertSame('some_queue', JobChainingTestThirdJob::$usedQueue);
  $this->assertSame('sync1', JobChainingTestThirdJob::$usedConnection);
}

public function testChainJobsUseOwnConfig()
{
  JobChainingTestFirstJob::dispatch()->allOnQueue('some_queue')->allOnConnection('sync1')->chain([
    (new JobChainingTestSecondJob)->onQueue('another_queue')->onConnection('sync2'),
    new JobChainingTestThirdJob,
  ]);

  $this->assertSame('some_queue', JobChainingTestFirstJob::$usedQueue);
  $this->assertSame('sync1', JobChainingTestFirstJob::$usedConnection);

  $this->assertSame('another_queue', JobChainingTestSecondJob::$usedQueue);
  $this->assertSame('sync2', JobChainingTestSecondJob::$usedConnection);

  $this->assertSame('some_queue', JobChainingTestThirdJob::$usedQueue);
  $this->assertSame('sync1', JobChainingTestThirdJob::$usedConnection);
}

public function testChainJobsUseDefaultConfig()
{
  JobChainingTestFirstJob::dispatch()->onQueue('some_queue')->onConnection('sync1')->chain([
    (new JobChainingTestSecondJob)->onQueue('another_queue')->onConnection('sync2'),
    new JobChainingTestThirdJob,
  ]);

  $this->assertSame('some_queue', JobChainingTestFirstJob::$usedQueue);
  $this->assertSame('sync1', JobChainingTestFirstJob::$usedConnection);

  $this->assertSame('another_queue', JobChainingTestSecondJob::$usedQueue);
  $this->assertSame('sync2', JobChainingTestSecondJob::$usedConnection);

  $this->assertNull(JobChainingTestThirdJob::$usedQueue);
  $this->assertNull(JobChainingTestThirdJob::$usedConnection);
}
```



### model

`Model::on` 设置model的连接

两个不同连接的model 不能别打包序列化 详见`SerializesModels`



如果序列化之前 model 存在，之后删除了，那么反序列化时，也就不会有了，因为反序列化会去查询一次。

```php
public function testItCanUnserializeACollectionInCorrectOrderAndHandleDeletedModels()
{
  ModelSerializationTestUser::create(['email' => '2@laravel.com']);
  ModelSerializationTestUser::create(['email' => '3@laravel.com']);
  ModelSerializationTestUser::create(['email' => '1@laravel.com']);

  $serialized = serialize(new CollectionSerializationTestClass(
    ModelSerializationTestUser::orderByDesc('email')->get()
  ));

  ModelSerializationTestUser::where(['email' => '2@laravel.com'])->delete();

  $unserialized = unserialize($serialized);

  $this->assertCount(2, $unserialized->users);

  $this->assertEquals($unserialized->users->first()->email, '3@laravel.com');
  $this->assertEquals($unserialized->users->last()->email, '1@laravel.com');
}
```



### route

路由绑定

```php
public function testRouteBindingsAreProperlySaved()
{
  $this->routeCollection->add($this->newRoute('GET', 'posts/{post:slug}/show', [
    'uses' => 'FooController@index',
    'prefix' => 'profile/{user:username}',
    'as' => 'foo',
  ]));

  $route = $this->collection()->getByName('foo');

  $this->assertSame('profile/{user}/posts/{post}/show', $route->uri());
  $this->assertSame(['user' => 'username', 'post' => 'slug'], $route->bindingFields());
}
```

设置 fallback 是get类型。本质是 prefix/*,不同路由组可以设置不同的fallback

```php
public function testFallbackWithPrefix()
{
  Route::group(['prefix' => 'prefix'], function () {
    Route::fallback(function () {
      return response('fallback', 404);
    });

    Route::get('one', function () {
      return 'one';
    });
  });

  $this->assertStringContainsString('one', $this->get('/prefix/one')->getContent());
  $this->assertStringContainsString('fallback', $this->get('/prefix/non-existing')->getContent());
  $this->assertStringContainsString('fallback', $this->get('/prefix/non-existing/with/multiple/segments')->getContent());
  $this->assertStringContainsString('Not Found', $this->get('/non-existing')->getContent());
}
```

respondWithRoute 可以从一个路由调用另一个路由

```php
public function testRespondWithNamedFallbackRoute()
{
  Route::fallback(function () {
    return response('fallback', 404);
  })->name('testFallbackRoute');

  Route::get('one', function () {
    return Route::respondWithRoute('testFallbackRoute');
  });

  $this->assertStringContainsString('fallback', $this->get('/non-existing')->getContent());
  $this->assertStringContainsString('fallback', $this->get('/one')->getContent());
}
```

api resources 

```php
Route::apiResources([
  'tests' => ApiResourceTestController::class,
  'tasks' => ApiResourceTaskController::class,
]);
```

带签名的路由

```php
public function testSigningUrl()
{
  Route::get('/foo/{id}', function (Request $request, $id) {
    return $request->hasValidSignature() ? 'valid' : 'invalid';
  })->name('foo');

  $this->assertIsString($url = URL::signedRoute('foo', ['id' => 1]));
  $this->assertSame('valid', $this->get($url)->original);
}
```

mock对象还可以用继承的方式

```php
Session::extend('fake-null', function () use ($handler) {
  return $handler;
});
```





### validate

exists 预发可以验证数组，底层用whereIN

```php
public function testExists()
{
  $validator = $this->getValidator(['first_name' => ['John', 'Jim']], ['first_name' => 'exists:users']);
  $this->assertFalse($validator->passes());
}
```



自己设置 attribute

```php
public function testImplicitAttributeFormatting()
{
  $translator = new Translator(new ArrayLoader, 'en');
  $translator->addLines(['validation.string' => ':attribute must be a string!'], 'en');
  $validator = new Validator($translator, [['name' => 1]], ['*.name' => 'string']);

  $validator->setImplicitAttributesFormatter(function ($attribute) {
    [$line, $attribute] = explode('.', $attribute);

    return sprintf('%s at line %d', $attribute, $line + 1);
  });

  $validator->passes();

  $this->assertSame('name at line 1 must be a string!', $validator->getMessageBag()->all()[0]);
}
```



### pipline

传参数

```php
public function testPipelineUsageWithParameters()
{
  $parameters = ['one', 'two'];

  $result = (new Pipeline(new Container))
    ->send('foo')
    ->through(PipelineTestParameterPipe::class.':'.implode(',', $parameters))
    ->then(function ($piped) {
      return $piped;
    });

  $this->assertSame('foo', $result);
  $this->assertEquals($parameters, $_SERVER['__test.pipe.parameters']);

  unset($_SERVER['__test.pipe.parameters']);
}
```





Arr Support

```php
public function testCrossJoin()
{
  // Single dimension
  $this->assertSame(
    [[1, 'a'], [1, 'b'], [1, 'c']],
    Arr::crossJoin([1], ['a', 'b', 'c'])
  );

  // Square matrix
  $this->assertSame(
    [[1, 'a'], [1, 'b'], [2, 'a'], [2, 'b']],
    Arr::crossJoin([1, 2], ['a', 'b'])
  );

  // Rectangular matrix
  $this->assertSame(
    [[1, 'a'], [1, 'b'], [1, 'c'], [2, 'a'], [2, 'b'], [2, 'c']],
    Arr::crossJoin([1, 2], ['a', 'b', 'c'])
  );

  // 3D matrix
  $this->assertSame(
    [
      [1, 'a', 'I'], [1, 'a', 'II'], [1, 'a', 'III'],
      [1, 'b', 'I'], [1, 'b', 'II'], [1, 'b', 'III'],
      [2, 'a', 'I'], [2, 'a', 'II'], [2, 'a', 'III'],
      [2, 'b', 'I'], [2, 'b', 'II'], [2, 'b', 'III'],
    ],
    Arr::crossJoin([1, 2], ['a', 'b'], ['I', 'II', 'III'])
  );

  // With 1 empty dimension
  $this->assertEmpty(Arr::crossJoin([], ['a', 'b'], ['I', 'II', 'III']));
  $this->assertEmpty(Arr::crossJoin([1, 2], [], ['I', 'II', 'III']));
  $this->assertEmpty(Arr::crossJoin([1, 2], ['a', 'b'], []));

  // With empty arrays
  $this->assertEmpty(Arr::crossJoin([], [], []));
  $this->assertEmpty(Arr::crossJoin([], []));
  $this->assertEmpty(Arr::crossJoin([]));

  // Not really a proper usage, still, test for preserving BC
  $this->assertSame([[]], Arr::crossJoin());
}
```

判断一个数组是否是关联数组

```php
public function testIsAssoc()
{
  $this->assertTrue(Arr::isAssoc(['a' => 'a', 0 => 'b']));
  $this->assertTrue(Arr::isAssoc([1 => 'a', 0 => 'b']));
  $this->assertTrue(Arr::isAssoc([1 => 'a', 2 => 'b']));
  $this->assertFalse(Arr::isAssoc([0 => 'a', 1 => 'b']));
  $this->assertFalse(Arr::isAssoc(['a', 'b']));
}
```



Arr pluck 另一个写法

```php
public function testPluckWithArrayValue()
{
  $array = [
    ['developer' => ['name' => 'Taylor']],
    ['developer' => ['name' => 'Abigail']],
  ];
  $array = Arr::pluck($array, ['developer', 'name']);
  $this->assertEquals(['Taylor', 'Abigail'], $array);
}

public function testPluckWithKeys()
{
  $array = [
    ['name' => 'Taylor', 'role' => 'developer'],
    ['name' => 'Abigail', 'role' => 'developer'],
  ];

  $test1 = Arr::pluck($array, 'role', 'name');
  $test2 = Arr::pluck($array, null, 'name');

  $this->assertEquals([
    'Taylor' => 'developer',
    'Abigail' => 'developer',
  ], $test1);

  $this->assertEquals([
    'Taylor' => ['name' => 'Taylor', 'role' => 'developer'],
    'Abigail' => ['name' => 'Abigail', 'role' => 'developer'],
  ], $test2);
}
```



Arr query

```php
public function testQuery()
{
  $this->assertSame('', Arr::query([]));
  $this->assertSame('foo=bar', Arr::query(['foo' => 'bar']));
  $this->assertSame('foo=bar&bar=baz', Arr::query(['foo' => 'bar', 'bar' => 'baz']));
  $this->assertSame('foo=bar&bar=1', Arr::query(['foo' => 'bar', 'bar' => true]));
  $this->assertSame('foo=bar', Arr::query(['foo' => 'bar', 'bar' => null]));
  $this->assertSame('foo=bar&bar=', Arr::query(['foo' => 'bar', 'bar' => '']));
}
```





```php
public function testDiffAssoc($collection)
{
  $c1 = new $collection(['id' => 1, 'first_word' => 'Hello', 'not_affected' => 'value']);
  $c2 = new $collection(['id' => 123, 'foo_bar' => 'Hello', 'not_affected' => 'value']);
  $this->assertEquals(['id' => 1, 'first_word' => 'Hello'], $c1->diffAssoc($c2)->all());
}
```



```php
public function testJoin($collection)
{
  $this->assertSame('a, b, c', (new $collection(['a', 'b', 'c']))->join(', '));

  $this->assertSame('a, b and c', (new $collection(['a', 'b', 'c']))->join(', ', ' and '));

  $this->assertSame('a and b', (new $collection(['a', 'b']))->join(', ', ' and '));

  $this->assertSame('a', (new $collection(['a']))->join(', ', ' and '));

  $this->assertSame('', (new $collection([]))->join(', ', ' and '));
}
```

Implode

```php
public function testImplode($collection)
{
  $data = new $collection([['name' => 'taylor', 'email' => 'foo'], ['name' => 'dayle', 'email' => 'bar']]);
  $this->assertSame('foobar', $data->implode('email'));
  $this->assertSame('foo,bar', $data->implode('email', ','));

  $data = new $collection(['taylor', 'dayle']);
  $this->assertSame('taylordayle', $data->implode(''));
  $this->assertSame('taylor,dayle', $data->implode(','));
}
```



自定义高阶函数 higherOrder

```php
/**
     * @dataProvider collectionClassProvider
     */
public function testCanAddMethodsToProxy($collection)
{
  $collection::macro('adults', function ($callback) {
    return $this->filter(function ($item) use ($callback) {
      return $callback($item) >= 18;
    });
  });

  $collection::proxy('adults');

  $c = new $collection([['age' => 3], ['age' => 12], ['age' => 18], ['age' => 56]]);

  $this->assertSame([['age' => 18], ['age' => 56]], $c->adults->age->values()->all());
}

```



```php
public function testMapInto($collection)
{
  $data = new $collection([
    'first', 'second',
  ]);

  $data = $data->mapInto(TestCollectionMapIntoObject::class);

  $this->assertSame('first', $data->get(0)->value);
  $this->assertSame('second', $data->get(1)->value);
}
```

groupBy

```php

/**
     * @dataProvider collectionClassProvider
     */
public function testGroupByClosureWhereItemsHaveMultipleGroups($collection)
{
  $data = new $collection([
    ['user' => 1, 'roles' => ['Role_1', 'Role_3']],
    ['user' => 2, 'roles' => ['Role_1', 'Role_2']],
    ['user' => 3, 'roles' => ['Role_1']],
  ]);

  $result = $data->groupBy(function ($item) {
    return $item['roles'];
  });

  $expected_result = [
    'Role_1' => [
      ['user' => 1, 'roles' => ['Role_1', 'Role_3']],
      ['user' => 2, 'roles' => ['Role_1', 'Role_2']],
      ['user' => 3, 'roles' => ['Role_1']],
    ],
    'Role_2' => [
      ['user' => 2, 'roles' => ['Role_1', 'Role_2']],
    ],
    'Role_3' => [
      ['user' => 1, 'roles' => ['Role_1', 'Role_3']],
    ],
  ];

  $this->assertEquals($expected_result, $result->toArray());
}


public function testGroupByMultiLevelAndClosurePreservingKeys($collection)
{
  $data = new $collection([
    10 => ['user' => 1, 'skilllevel' => 1, 'roles' => ['Role_1', 'Role_3']],
    20 => ['user' => 2, 'skilllevel' => 1, 'roles' => ['Role_1', 'Role_2']],
    30 => ['user' => 3, 'skilllevel' => 2, 'roles' => ['Role_1']],
    40 => ['user' => 4, 'skilllevel' => 2, 'roles' => ['Role_2']],
  ]);

  $result = $data->groupBy([
    'skilllevel',
    function ($item) {
      return $item['roles'];
    },
  ], true);

  $expected_result = [
    1 => [
      'Role_1' => [
        10 => ['user' => 1, 'skilllevel' => 1, 'roles' => ['Role_1', 'Role_3']],
        20 => ['user' => 2, 'skilllevel' => 1, 'roles' => ['Role_1', 'Role_2']],
      ],
      'Role_3' => [
        10 => ['user' => 1, 'skilllevel' => 1, 'roles' => ['Role_1', 'Role_3']],
      ],
      'Role_2' => [
        20 => ['user' => 2, 'skilllevel' => 1, 'roles' => ['Role_1', 'Role_2']],
      ],
    ],
    2 => [
      'Role_1' => [
        30 => ['user' => 3, 'skilllevel' => 2, 'roles' => ['Role_1']],
      ],
      'Role_2' => [
        40 => ['user' => 4, 'skilllevel' => 2, 'roles' => ['Role_2']],
      ],
    ],
  ];

  $this->assertEquals($expected_result, $result->toArray());
}

```

contains



```php
public function testContains($collection)
{
  $c = new $collection([1, 3, 5]);

  $this->assertTrue($c->contains(1));
  $this->assertTrue($c->contains('1'));
  $this->assertFalse($c->contains(2));
  $this->assertFalse($c->contains('2'));

  $c = new $collection(['1']);
  $this->assertTrue($c->contains('1'));
  $this->assertTrue($c->contains(1));

  $c = new $collection([null]);
  $this->assertTrue($c->contains(false));
  $this->assertTrue($c->contains(null));
  $this->assertTrue($c->contains([]));
  $this->assertTrue($c->contains(0));
  $this->assertTrue($c->contains(''));

  $c = new $collection([0]);
  $this->assertTrue($c->contains(0));
  $this->assertTrue($c->contains('0'));
  $this->assertTrue($c->contains(false));
  $this->assertTrue($c->contains(null));

  $this->assertTrue($c->contains(function ($value) {
    return $value < 5;
  }));
  $this->assertFalse($c->contains(function ($value) {
    return $value > 5;
  }));

  $c = new $collection([['v' => 1], ['v' => 3], ['v' => 5]]);

  $this->assertTrue($c->contains('v', 1));
  $this->assertFalse($c->contains('v', 2));

  $c = new $collection(['date', 'class', (object) ['foo' => 50]]);

  $this->assertTrue($c->contains('date'));
  $this->assertTrue($c->contains('class'));
  $this->assertFalse($c->contains('foo'));

  $c = new $collection([['a' => false, 'b' => false], ['a' => true, 'b' => false]]);

  $this->assertTrue($c->contains->a);
  $this->assertFalse($c->contains->b);

  $c = new $collection([
    null, 1, 2,
  ]);

  $this->assertTrue($c->contains(function ($value) {
    return is_null($value);
  }));
}
```



```php
public function testContainsWithOperator($collection)
{
  $c = new $collection([['v' => 1], ['v' => 3], ['v' => '4'], ['v' => 5]]);

  $this->assertTrue($c->contains('v', '=', 4));
  $this->assertTrue($c->contains('v', '==', 4));
  $this->assertFalse($c->contains('v', '===', 4));
  $this->assertTrue($c->contains('v', '>', 4));
}
```

sortBy

```php
public function testValueRetrieverAcceptsDotNotation($collection)
{
  $c = new $collection([
    (object) ['id' => 1, 'foo' => ['bar' => 'B']], (object) ['id' => 2, 'foo' => ['bar' => 'A']],
  ]);

  $c = $c->sortBy('foo.bar');
  $this->assertEquals([2, 1], $c->pluck('id')->all());
}
```

reject

```php
public function testRejectWithoutAnArgumentRemovesTruthyValues($collection)
{
  $data1 = new $collection([
    false,
    true,
    new $collection(),
    0,
  ]);
  $this->assertSame([0 => false, 3 => 0], $data1->reject()->all());

  $data2 = new $collection([
    'a' => true,
    'b' => true,
    'c' => true,
  ]);
  $this->assertTrue(
    $data2->reject()->isEmpty()
  );
}

```

tappable

```php

class TappableClass
{
    use Tappable;

    private $name;

    public static function make()
    {
        return new static;
    }

    public function setName($name)
    {
        $this->name = $name;
    }

    public function getName()
    {
        return $this->name;
    }
}
```


