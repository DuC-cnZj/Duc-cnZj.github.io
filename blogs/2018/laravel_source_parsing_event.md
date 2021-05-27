---
title: Laravel 源码解析之 Event
date: '2021-05-12 12:54:14'
sidebar: true
categories:
 - laravel
tags:
 - 源码解析
 - Event
publish: true
---

laravel事件是如何运作的，我们先从事件的注册开始说起

事件在很早就注册并且实例化了，因为路由的实例依赖事件实例

```php

class Application extends Container implements ApplicationContract, HttpKernelInterface
{
	 ...
	 
    /**
     * Create a new Illuminate application instance.
     *
     * @param  string|null  $basePath
     * @return void
     */
    public function __construct($basePath = null)
    {
        if ($basePath) {
            $this->setBasePath($basePath);
        }

        $this->registerBaseBindings();

        $this->registerBaseServiceProviders();

        $this->registerCoreContainerAliases();
    }
    
    
class EventServiceProvider extends ServiceProvider
{
    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        # 这里可以看出 之后所有的事件实例都是这个 Dispatcher 类的实例
        $this->app->singleton('events', function ($app) {
            return (new Dispatcher($app))->setQueueResolver(function () use ($app) {
                return $app->make(QueueFactoryContract::class);
            });
        });
    }
```

事件在kernel实例化之前就存在了，之后再kernel调用handle的时候，去注册，也就是 bootstrap() 时 ，最后一步 

BootProviders 调用了所有serverProvider的boot方法

```php
protected $bootstrappers = [
    \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class,
    \Illuminate\Foundation\Bootstrap\LoadConfiguration::class,
    \Illuminate\Foundation\Bootstrap\HandleExceptions::class,
    \Illuminate\Foundation\Bootstrap\RegisterFacades::class,
    \Illuminate\Foundation\Bootstrap\RegisterProviders::class,
    \Illuminate\Foundation\Bootstrap\BootProviders::class,
];
```

我们来看 EventServiceProvider 的 boot

```php

class EventServiceProvider extends ServiceProvider
{
    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
        DucEvent::class => [
            DucListener::class,
        ],
        '*' => [
            DucListener::class
        ]
    ];

    /**
     * Register any events for your application.
     *
     * @return void
     */
    public function boot()
    {
        parent::boot();

        //
    }
}

public function boot()
{
    # $this->listens() 返回 listen 属性
    # 可以看到这里它用了 event facade 的 listen，也就是 Dispatch 类的listen
    foreach ($this->listens() as $event => $listeners) {
        foreach ($listeners as $listener) {
            Event::listen($event, $listener);
        }
    }

    foreach ($this->subscribe as $subscriber) {
        Event::subscribe($subscriber);
    }
}


# Dispatch
# 可以看到下面打印中的信息，就是一个event名对应一个处理的闭包
public function listen($events, $listener)
{
    foreach ((array) $events as $event) {
        # 注意这里。框架回去匹配通配符 * 
        if (Str::contains($event, '*')) {
            $this->setupWildcardListen($event, $listener);
        } else {
            $this->listeners[$event][] = $this->makeListener($listener);
        }
    }
}

```

```php
# Dispatch listeners属性
array:23 [▼
  "Illuminate\Cache\Events\CacheHit" => array:1 [▼
    0 => Closure {#164 ▶}
  ]
  "Illuminate\Cache\Events\CacheMissed" => array:1 [▶]
  "Illuminate\Cache\Events\KeyWritten" => array:1 [▶]
  "Illuminate\Cache\Events\KeyForgotten" => array:1 [▶]
  "Illuminate\Console\Events\CommandFinished" => array:1 [▶]
  "Illuminate\Log\Events\MessageLogged" => array:2 [▶]
  "Illuminate\Queue\Events\JobProcessed" => array:2 [▶]
  "Illuminate\Queue\Events\JobFailed" => array:2 [▶]
  "Illuminate\Mail\Events\MessageSent" => array:1 [▶]
  "Illuminate\Notifications\Events\NotificationSent" => array:1 [▶]
  "Illuminate\Database\Events\QueryExecuted" => array:1 [▶]
  "Illuminate\Redis\Events\CommandExecuted" => array:1 [▶]
  "Illuminate\Foundation\Http\Events\RequestHandled" => array:1 [▶]
  "Illuminate\Console\Events\CommandStarting" => array:1 [▶]
  "Illuminate\Queue\Events\JobProcessing" => array:1 [▶]
  "Illuminate\Queue\Events\JobExceptionOccurred" => array:1 [▶]
  "Illuminate\Foundation\Events\LocaleUpdated" => array:1 [▶]
  "eloquent.creating: Spatie\MediaLibrary\Models\Media" => array:1 [▶]
  "eloquent.updating: Spatie\MediaLibrary\Models\Media" => array:1 [▶]
  "eloquent.updated: Spatie\MediaLibrary\Models\Media" => array:1 [▶]
  "eloquent.deleted: Spatie\MediaLibrary\Models\Media" => array:1 [▶]
  "Illuminate\Auth\Events\Registered" => array:1 [▶]
  "App\Events\DucEvent" => array:1 [▶]
]
```

当注册完成之后，启动之前，框架本身还会触发一个名为 `bootstrapped: Illuminate\Foundation\Bootstrap\BootProviders` 的启动事件，

```php
public function bootstrapWith(array $bootstrappers)
{
    $this->hasBeenBootstrapped = true;

    foreach ($bootstrappers as $bootstrapper) {
        # fire 方法其实就是 dispatch 方法，不过第三个参数为 false，虽然 Dispatch 默认也是 false
        # 上面说到再启动之前触发的事件只有 bootstrapped: Illuminate\Foundation\Bootstrap\BootProviders 这一个，可这里明明还有很多 bootstrapping 和 别的，那这是为什么没有触发呢？原因在于用户自定义的 listener 此时还没有注册！
        # 如果你一定要监听系统所有的事件包括启动之前的，那么请你在 index.php $kernel->handle 之前 加入
         # $app['events']->listen('*', \App\Listeners\DucListener::class);

        $this['events']->fire('bootstrapping: '.$bootstrapper, [$this]);

        $this->make($bootstrapper)->bootstrap($this);

        $this['events']->fire('bootstrapped: '.$bootstrapper, [$this]);
    }
}
```

注册完之后你的 Dispatch 类中有了对应的listener之后，你就可以在控制器或者别的地方触发事件，底层调用了 dispatch 方法

```php
public function dispatch($event, $payload = [], $halt = false)
{

    # 格式化event和payload变量，使其变成 ['App\Event\DucEvent', ['xxx']] 格式
    [$event, $payload] = $this->parseEventAndPayload(
        $event, $payload
    );
    
    # 是否需要广播
    if ($this->shouldBroadcast($payload)) {
        $this->broadcastEvent($payload[0]);
    }

    $responses = [];

    # 循环调用对应的事件监听类
    foreach ($this->getListeners($event) as $listener) {
        # 执行对应的监听方法
        $response = $listener($event, $payload);

		
        # 这里的 halt 参数如果设置为 true 并且 listener 中有返回值，那么它将不再执行后面的 listener 然后返回对应的返回值
        if ($halt && ! is_null($response)) {
            return $response;
        }


        # 这儿的意思是，当你一个事件触发多个时间监听器的时候，你可以在某一个监听器中的 handle 方法中返回 false，那么框架就不会继续执行剩下的监听器了，但注意如果是队列化的，这里将不起作用
        if ($response === false) {
            break;
        }

        $responses[] = $response;
    }

    return $halt ? null : $responses;
}

public function getListeners($eventName)
{
    $listeners = $this->listeners[$eventName] ?? [];

    # 这里框架用正则调用你的通配符*去配对事件名称，如果匹配上就加入 listeners 中
    $listeners = array_merge(
        $listeners,
        $this->wildcardsCache[$eventName] ?? $this->getWildcardListeners($eventName)
    );

    # 返回所有匹配到的监听类(闭包)
    return class_exists($eventName, false)
        ? $this->addInterfaceListeners($eventName, $listeners)
        : $listeners;
}
```

```php
# 实现队列接口，队列化发送的原理在此
public function createClassListener($listener, $wildcard = false)
{
    return function ($event, $payload) use ($listener, $wildcard) {
        if ($wildcard) {
            return call_user_func($this->createClassCallable($listener), $event, $payload);
        }

        return call_user_func_array(
            $this->createClassCallable($listener), $payload
        );
    };
}

# 这里可以看到对于有通配符的事件 比如 event.*.bootstraped 该方法会给对应的 监听器 Listener 传两个参数
# $event, $payload, 而对于具体的事件，比如 event.user.bootstraped，它只会传一个参数 $payload
public function createClassListener($listener, $wildcard = false)
{
    return function ($event, $payload) use ($listener, $wildcard) {
        if ($wildcard) {
            return call_user_func($this->createClassCallable($listener), $event, $payload);
        }

        return call_user_func_array(
            $this->createClassCallable($listener), $payload
        );
    };
}


# createClassCallable 中这里，看出代码判断了是否使用队列
if ($this->handlerShouldBeQueued($class)) {
    return $this->createQueuedHandlerCallable($class, $method);
}

# 从这里可以看出，即使实现了 ShouldQueue 接口也可以不让它队列化执行
# 只需要在 listener 中 添加一个名为 shouldQueue 的方法返回 false
protected function handlerWantsToBeQueued($class, $arguments)
{
    if (method_exists($class, 'shouldQueue')) {
        return $this->container->make($class)->shouldQueue($arguments[0]);
    }

    return true;
}

protected function handlerShouldBeQueued($class)
{
    try {
        return (new ReflectionClass($class))->implementsInterface(
            ShouldQueue::class
        );
    } catch (Exception $e) {
        return false;
    }
}
```