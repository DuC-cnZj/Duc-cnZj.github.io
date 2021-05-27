---
title: Laravel 自定义request类如何自动实现表单验证
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - laravel
tags:
 - 源码解读
publish: true
---


> 如下情况：当自定义表单验证类之后，框架会调用表单验证方法进行验证，框架是如何自动发现request类，并且完成表单验证的？

```php
// TestRequest

// ...
class TestRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }


    public function rules()
    {
        return [
            'name' =>'required'
        ];
    }
}

// HomeController
// ... 
public function store (TestRequest $request) {
  // ...
}
```



探索源码

我们发现在 `config/app.php` 中默认配置了 `FoundationServiceProvider` , 大家都知道laravel的服务都是通过ServiceProvider来提供的，而 `FoundationServiceProvider`中提供了表单验证的服务。

```php
protected $providers = [
  FormRequestServiceProvider::class,
];

public function register()
{
  # 注册了 FormRequestServiceProvider
  parent::register();

  $this->registerRequestValidation();
  $this->registerRequestSignatureValidation();
}
```

FormRequestServiceProvider.php

```php
public function boot()
{
  $this->app->afterResolving(ValidatesWhenResolved::class, function ($resolved) {
    $resolved->validateResolved();
  });

  $this->app->resolving(FormRequest::class, function ($request, $app) {
    $request = FormRequest::createFrom($app['request'], $request);

    $request->setContainer($app)->setRedirector($app->make(Redirector::class));
  });
}
```

来看看 boot 中的两个方法

```php
public function afterResolving($abstract, Closure $callback = null)
{
  if (is_string($abstract)) {
    $abstract = $this->getAlias($abstract);
  }

  // 如果只传第一个参数并且是闭包，那么将闭包加入 app 实例的 globalAfterResolvingCallbacks 属性
  // 否则 加入 afterResolvingCallbacks 属性
  if ($abstract instanceof Closure && is_null($callback)) {
    $this->globalAfterResolvingCallbacks[] = $abstract;
  } else {
    $this->afterResolvingCallbacks[$abstract][] = $callback;
  }
}
```

那么这两个属性都是干嘛用的呢

探索源码可以发现，框架用容器实例化的时候用到了这两个属性

来看 `app()->make()` => `resolve()`, 第三个属性默认是 true，来看它有什么用，可以发现这个参数在下面代码中触发了`$this->fireResolvingCallbacks($abstract, $object);`

```php
protected function resolve($abstract, $parameters = [], $raiseEvents = true)
{
    $abstract = $this->getAlias($abstract);

    $needsContextualBuild = ! empty($parameters) || ! is_null(
        $this->getContextualConcrete($abstract)
    );

    if (isset($this->instances[$abstract]) && ! $needsContextualBuild) {
        return $this->instances[$abstract];
    }

    $this->with[] = $parameters;

    $concrete = $this->getConcrete($abstract);

    if ($this->isBuildable($concrete, $abstract)) {
        $object = $this->build($concrete);
    } else {
        $object = $this->make($concrete);
    }

    foreach ($this->getExtenders($abstract) as $extender) {
        $object = $extender($object, $this);
    }

    if ($this->isShared($abstract) && ! $needsContextualBuild) {
        $this->instances[$abstract] = $object;
    }

    if ($raiseEvents) {
        $this->fireResolvingCallbacks($abstract, $object);
    }

    $this->resolved[$abstract] = true;

    array_pop($this->with);

    return $object;
}
```

继续深入，`globalResolvingCallbacks` 浮出水面

```php
protected function fireResolvingCallbacks($abstract, $object)
{
  $this->fireCallbackArray($object, $this->globalResolvingCallbacks);

  $this->fireCallbackArray(
    $object, $this->getCallbacksForType($abstract, $object, $this->resolvingCallbacks)
  );

  $this->fireAfterResolvingCallbacks($abstract, $object);
}
```

可以看到 globalResolvingCallbacks 在每个实例解析后触发调用, 传入了实例本身和app容器实例,只要通过容器解析的class都会触发该属性的闭包方法

```php
protected function fireCallbackArray($object, array $callbacks)
{
  foreach ($callbacks as $callback) {
    $callback($object, $this);
  }
}
```

接下来再看

```php
// 当容器实例化 App\Http\Requests 类时，这里传的三个参数就是
// App\Http\Requests, App\Http\Requests 实例, 以及 resolvingCallbacks 属性
// 例如
//  # resolvingCallbacks: array:1 [▼
//    "App\Contract\AfterResolve" => array:1 [▼
//      0 => Closure() {#130 ▶}
//    ]
//  ]
protected function getCallbacksForType($abstract, $object, array $callbacksPerType)
{
  $results = [];

  foreach ($callbacksPerType as $type => $callbacks) {
    // 这里很关键 如果实例本身类就是$abstract 或者实例是 注册的$type的子类，都会触发回调
    if ($type === $abstract || $object instanceof $type) {
      $results = array_merge($results, $callbacks);
    }
  }

  return $results;
}
```

上面的是 resolving 时触发的，框架还挺了解析后触发的回调

```php

    /**
     * Register a new resolving callback.
     *
     * @param  \Closure|string  $abstract
     * @param  \Closure|null  $callback
     * @return void
     */
    public function resolving($abstract, Closure $callback = null);

    /**
     * Register a new after resolving callback.
     *
     * @param  \Closure|string  $abstract
     * @param  \Closure|null  $callback
     * @return void
     */
    public function afterResolving($abstract, Closure $callback = null);
```

而我们要探索的表单就是属于解析后的闭包调用

```php
public function boot()
{
  // 和上面的解析时差不多
  $this->app->afterResolving(ValidatesWhenResolved::class, function ($resolved) {
    $resolved->validateResolved();
  });

  $this->app->resolving(FormRequest::class, function ($request, $app) {
    $request = FormRequest::createFrom($app['request'], $request);

    $request->setContainer($app)->setRedirector($app->make(Redirector::class));
  });
}
```

可以看到我们生成的 TestRequest 实现了 `ValidatesWhenResolved::class` 接口，也可以触发回调，所以当 TestRequest 通过容器实例解析好了后就触发了 `    $resolved->validateResolved();`，这时候就触发了表单验证

```php
public function validateResolved()
{
  $this->prepareForValidation();

  if (! $this->passesAuthorization()) {
    $this->failedAuthorization();
  }

  $instance = $this->getValidatorInstance();

  if ($instance->fails()) {
    $this->failedValidation($instance);
  }

  $this->passedValidation();
}
```

你可以在ServiceProvider中自己测一下

```php
public function boot()
{
  // AfterResolve::class 子类或者自己解析后才触发
  $this->app->afterResolving(AfterResolve::class, function ($obj, $app) {
    dump($obj, $app);
  });

  // 所以类解析后都会触发
  $this->app->afterResolving(function ($obj,$app) {
    dump($obj);
  });
}
```

触发顺序

1. globalResolvingCallbacks
2. resolvingCallbacks
3. globalAfterResolvingCallbacks
4. afterResolvingCallbacks

除了顺序之外好像也没啥明显区别🧐