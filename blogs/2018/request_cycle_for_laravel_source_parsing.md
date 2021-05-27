---
title: laravel源码解析之请求周期
date: '2021-05-12 12:54:14'
sidebar: true
categories:
 - laravel
tags:
 - 源码解析
 - 请求周期
publish: true
---



## 入口文件 indx.php

```php
<?php

# 系统定义了一个开始时间
define('LARAVEL_START', microtime(true));

# composer 类自动加载
require __DIR__.'/../vendor/autoload.php';

# 系统初始化
$app = require_once __DIR__.'/../bootstrap/app.php';

# 正式实例化类，在这之前的绑定和别名种种都是存下了抽象类或者别名与其生成类的闭包函数的数组，没有正式实例化，直到执行下面这句，make 返回实例化的 Kernel 类，而 kernel 类需要用到路由类，路由类需要传递一个实例化的 Event 类，所以到此为止实例化的只有三个类，Kernel、Router 和 Event。打印出来的结果也证实了这一点。
resolved: array:4 [▼
   "events" => true
   "router" => true
   "App\Http\Kernel" => true
   "Illuminate\Contracts\Http\Kernel" => true
]
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

# 最后，内核处理相应的请求
$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);

# 将请求发送给客户端
$response->send();

# 最后处理下结束中间件，注意如果在request中的terminate方法里面 dd()，或者echo什么的操作你是看不到的，因为响应已经在上一个方法中返回了，这里只能用 \Log 来记录。
$kernel->terminate($request, $response);

```

进入 `bootstrap/app.php`

```php
<?php

# 设置了框架的根路径
$app = new Illuminate\Foundation\Application(
    dirname(__DIR__)
);


$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);

return $app;

```

## 代码深入

```php
$app = new Illuminate\Foundation\Application(
	dirname(__DIR__)
);
```

上面代码做了以下的几件事

1. 设置app的根目录以及绑定其他文件夹的目录 `setBasePath->bindPathsInContainer` 

   1. 先来看看 Application 类中内置的属性

      1. ```php
           # 保证整个应用单一实例的单例，static 后期静态绑定
              	protected static $instance;
              
             # 记录运行到当前这一刻，已经实例化的类
             resolved: array:4 [▼
               "events" => true
               "router" => true
               "App\Http\Kernel" => true
               "Illuminate\Contracts\Http\Kernel" => true
             ]
              protected $resolved = [];
              
              	# 记录接口和类的生成闭包，以及是否共享
              protected $bindings = [];
              
              /**
               * The container's method bindings.
               *
               * @var array
               */
              protected $methodBindings = [];
              
              	# 存的都是具体的实例化类和系统路径
              protected $instances = [];
              
              'aliases' => [
                  "Illuminate\Foundation\Application" => "app"
                  "Illuminate\Contracts\Container\Container" => "app"
                  "Illuminate\Contracts\Foundation\Application" => "app"
                  "Psr\Container\ContainerInterface" => "app"
              ]
              protected $aliases = [];
              
              /**
               * The registered aliases keyed by the abstract name.
               *
               * @var array
               */
              protected $abstractAliases = [];
             /**
              * The extension closures for services.
              *
              * @var array
              */
             protected $extenders = [];
             
             /**
              * All of the registered tags.
              *
              * @var array
              */
             protected $tags = [];
    
            	# 用来临时储存当前需要实例化的类，实例化完就清空了
             buildStack: array:1 [▼
               0 => "App\Http\Kernel"
             ]
             protected $buildStack = [];
            	# 用来储存当前实例化的类所需要的参数，实例化完就清空了
             protected $with = [];
             /**
              * The contextual binding map.
              *
              * @var array
              */
             public $contextual = [];
             
             /**
              * All of the registered rebound callbacks.
              *
              * @var array
              */
             protected $reboundCallbacks = [];
             
             /**
              * All of the global resolving callbacks.
              *
              * @var array
              */
             protected $globalResolvingCallbacks = [];
             
             /**
              * All of the global after resolving callbacks.
              *
              * @var array
              */
             protected $globalAfterResolvingCallbacks = [];
             
             /**
              * All of the resolving callbacks by class type.
              *
              * @var array
              */
             protected $resolvingCallbacks = [];
             
             /**
              * All of the after resolving callbacks by class type.
              *
              * @var array
              */
             protected $afterResolvingCallbacks = [];
         ```

   2. 调用了 Application 的 `instance` 方法

   3. ```php
      # abstractAliases 中的格式类似这样
      "app" => array:4 [
          0 => "Illuminate\Foundation\Application"
          1 => "Illuminate\Contracts\Container\Container"
          2 => "Illuminate\Contracts\Foundation\Application"
          3 => "Psr\Container\ContainerInterface"
      ]
      
      public function instance($abstract, $instance)
      {
      #    dd($abstract, $instance); path , /var/www/html/app
          $this->removeAbstractAlias($abstract);
      
          $isBound = $this->bound($abstract);
      
          # 上面删除了 属性 abstractAliases 中，$abstract 对应的别名，这里要把 $abstract 键值也删除
          unset($this->aliases[$abstract]);
      
          # 删除之后重新绑定，即 $instance => ['path'=>'/var/www/html/app']
          $this->instances[$abstract] = $instance;
      
          if ($isBound) {
              # 取消bingings instances 和 aliases中的绑定
              $this->rebound($abstract);
          }
      
          return $instance;
      }
      ```

      ```php
      # $searched => path
      # 先去搜索 alias 里面的 path 键值，不存在则返回，存在则删除存在的值
      protected function removeAbstractAlias($searched)
      {
          if (! isset($this->aliases[$searched])) {
              return;
          }
      
          foreach ($this->abstractAliases as $abstract => $aliases) {
              foreach ($aliases as $index => $alias) {
                  if ($alias == $searched) {
                      unset($this->abstractAliases[$abstract][$index]);
                  }
              }
          }
      }
      ```

      ```php
      # $abstract => path
      # 去判断 bingings instances 和 aliases 中是否存在 path 键值
      # 显然这里返回 false
      public function bound($abstract)
      {
          return isset($this->bindings[$abstract]) ||
              isset($this->instances[$abstract]) ||
              $this->isAlias($abstract);
      }
      ```

      ```php
      protected function rebound($abstract)
      {
          # 这一步不贴代码了，它具体做了以下几个操作
          # 1. 去 alias 中找是否存在键值，
          # 2. 去看 buildStack[] 是否创建过这个实例
          $instance = $this->make($abstract);
      
          foreach ($this->getReboundCallbacks($abstract) as $callback) {
              call_user_func($callback, $this, $instance);
          }
      }
      ```

      ```php
      # 这个方法的递归主要是为了找到根实例
      # 因为 $aliases 中格式可能是这样的
      # 'aliases' => [
      #	'a' => 'b',
      # 	'b' => 'c',
      # ]
      # a 的别名是 b，b 才对应到 c，c才是最终能找到实例的别名类
      public function getAlias($abstract)
      {
          if (! isset($this->aliases[$abstract])) {
              return $abstract;
          }
      
          if ($this->aliases[$abstract] === $abstract) {
              throw new LogicException("[{$abstract}] is aliased to itself.");
          }
      
          return $this->getAlias($this->aliases[$abstract]);
      }
      ```

      tips

      ```php
      true || $a = 'a'; // 后半句不会执行
      echo $a; //  Undefined variable: a
      ```

      ```php
      public function __construct($basePath = null)
      {
          # 这一步绑定了所有目录
          array:9 [▼
            "path" => "/var/www/html/app"
            "path.base" => "/var/www/html"
            "path.lang" => "/var/www/html/resources/lang"
            "path.config" => "/var/www/html/config"
            "path.public" => "/var/www/html/public"
            "path.storage" => "/var/www/html/storage"
            "path.database" => "/var/www/html/database"
            "path.resources" => "/var/www/html/resources"
            "path.bootstrap" => "/var/www/html/bootstrap"
          ]
          if ($basePath) {
              $this->setBasePath($basePath);
          }
      
          # 这一步做了
          # 设置了static::$instance，保证了运行中的唯一单例
          # 绑定了 $instance['app'] = $this
          # 绑定了 $instance['Illuminate\Container\Container'] = $this
          # 绑定了 $instance['Illuminate\Foundation\PackageManifest'] = PackageManifest 实例(大概是启动的时候需要给配置文件做缓存，所以优先级比较高)
          array:12 [▼
            "path" => "/var/www/html/app"
            "path.base" => "/var/www/html"
            "path.lang" => "/var/www/html/resources/lang"
            "path.config" => "/var/www/html/config"
            "path.public" => "/var/www/html/public"
            "path.storage" => "/var/www/html/storage"
            "path.database" => "/var/www/html/database"
            "path.resources" => "/var/www/html/resources"
            "path.bootstrap" => "/var/www/html/bootstrap"
            "app" => Application {#2 ▶}
            "Illuminate\Container\Container" => Application {#2 ▶}
            "Illuminate\Foundation\PackageManifest" => PackageManifest {#4 ▶}
          ]
          $this->registerBaseBindings();
      
      	# 添加系统最基础最重要的服务
          $this->registerBaseServiceProviders();
                
      	# 注册系统核心服务 $aliases 和 $abstractAliases 属性改变了
          aliases => [
              "Illuminate\Foundation\Application" => "app"
      	    "Illuminate\Contracts\Container\Container" => "app"
          	"Illuminate\Contracts\Foundation\Application" => "app"
      	    "Psr\Container\ContainerInterface" => "app"
          ]
          abstractAliases: array:35 [▼
            "app" => array:4 [▼
                0 => "Illuminate\Foundation\Application"
                1 => "Illuminate\Contracts\Container\Container"
                2 => "Illuminate\Contracts\Foundation\Application"
                3 => "Psr\Container\ContainerInterface"
             ]
          $this->registerCoreContainerAliases();
      }
      ```

      ```php
      protected function registerBaseServiceProviders()
      {
          $this->register(new EventServiceProvider($this));
      
          $this->register(new LogServiceProvider($this));
      
          $this->register(new RoutingServiceProvider($this));
      }
      ```

      

      ```php
      public function register($provider, $force = false)
      {
          # getProvider 去获取 serviceProviders 数组中的 provider
          # 显然之前并没有注册过，return false
          if (($registered = $this->getProvider($provider)) && ! $force) {
              return $registered;
          }
      
      	# 这里传入的是实例，所以 false
          if (is_string($provider)) {
              $provider = $this->resolveProvider($provider);
          }
      
          # 调用 EventServiceProvider，LogServiceProvider， RoutingServiceProvider 的 register 方法，绑定了系统日志，事件和路由
           bindings: array:9 [▼
              "events" => array:2 [▶]
              "log" => array:2 [▶]
              "router" => array:2 [▶]
              "url" => array:2 [▶]
              "redirect" => array:2 [▶]
              "Psr\Http\Message\ServerRequestInterface" => array:2 [▼
                "concrete" => Closure {#14 ▶}
                "shared" => false
              ]
              "Psr\Http\Message\ResponseInterface" => array:2 [▶]
              "Illuminate\Contracts\Routing\ResponseFactory" => array:2 [▶]
              "Illuminate\Routing\Contracts\ControllerDispatcher" => array:2 [▶]
            ]
          if (method_exists($provider, 'register')) {
              $provider->register();
          }
                    
      	## 下面两个可以解释为什么在 serverProvider 中可以使用 $bindings 和 $singletons 绑定属性
      
      	# 解析provider中的 bingings 属性
          if (property_exists($provider, 'bindings')) {
              foreach ($provider->bindings as $key => $value) {
                  # bind 方法用到了反射，通过反射拿到对应实例的构造函数
                  $this->bind($key, $value);
              }
          }
                    
      	# 解析provider中的 singletons 属性
          if (property_exists($provider, 'singletons')) {
              foreach ($provider->singletons as $key => $value) {
                  $this->singleton($key, $value);
              }
          }
      
      	# 把解析过的 serviceProvider 添加到 $serviceProviders 和 $loadedProviders 属性中
         serviceProviders: array:3 [▼
          0 => EventServiceProvider {#6 ▶}
          1 => LogServiceProvider {#8 ▶}
          2 => RoutingServiceProvider {#10 ▶}
        ]
        loadedProviders: array:3 [▼
          "Illuminate\Events\EventServiceProvider" => true
          "Illuminate\Log\LogServiceProvider" => true
          "Illuminate\Routing\RoutingServiceProvider" => true
        ]
          $this->markAsRegistered($provider);
      
          # 当系统已经启动，则调用boot方法，只有延迟加载的provider才会走入这一步，也就是app中 deferredServices 中的provider
          if ($this->booted) {
              $this->bootProvider($provider);
          }
      
          return $provider;
      }
      ```

      ```php
      # EventServiceProvider
      public function register()
      {
          # $this->app 在这里把 Application 实例传进去了 -> $this->register(new EventServiceProvider($this));
          # 然后 在 ServiceProvider 的构造函数中把 实例赋值给了 app属性
          $this->app->singleton('events', function ($app) {
              return (new Dispatcher($app))->setQueueResolver(function () use ($app) {
                  return $app->make(QueueFactoryContract::class);
              });
          });
      }
      
      # LogServiceProvider
      public function register()
      {
          $this->app->singleton('log', function () {
              return new LogManager($this->app);
          });
      }
      ```

      ```php
      public function singleton($abstract, $concrete = null)
      {
          $this->bind($abstract, $concrete, true);
      }
      
      public function bind($abstract, $concrete = null, $shared = false)
      {
          # 如果已经绑定过instance，或存过 aliases 则删除
          $this->dropStaleInstances($abstract);
      
          if (is_null($concrete)) {
              $concrete = $abstract;
          }
      
          # 不是闭包则获取闭包
          if (! $concrete instanceof Closure) {
              $concrete = $this->getClosure($abstract, $concrete);
          }
      
          # 绑定实例
          bindings: array:1 [▼
           "events" => array:2 [▼
                "concrete" => Closure {#7 ▶} # 这里存了生成实例的闭包
                 "shared" => true
               ]
             ]
          $this->bindings[$abstract] = compact('concrete', 'shared');
      
      	# 最后还不忘解绑已经实例化的抽象属性              
          if ($this->resolved($abstract)) {
              $this->rebound($abstract);
          }
      }
      ```

2. 紧接着绑定了三个单例，高度解耦，把抽象和实例进行绑定

   1. ```php
      $app->singleton(
          Illuminate\Contracts\Http\Kernel::class,
          App\Http\Kernel::class
      );
      
      $app->singleton(
          Illuminate\Contracts\Console\Kernel::class,
          App\Console\Kernel::class
      );
      
      $app->singleton(
          Illuminate\Contracts\Debug\ExceptionHandler::class,
          App\Exceptions\Handler::class
      );
      ```

3. 然后拿出内核

   1. ```php
      # 上面进行了绑定，所以拿出来的就是 App\Http\Kernel::class 实例
      $kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
      ```

      `make` 方法最终会走到 `build`，这里注意框架内部用反射，拿到构造函数以及参数.

      ```php
      protected function resolve($abstract, $parameters = [])
          {
              $abstract = $this->getAlias($abstract);
              $needsContextualBuild = ! empty($parameters) || ! is_null(
                  $this->getContextualConcrete($abstract)
              );
      
      		# 判断是否已经实例化过类，有的话直接返回该实例
              if (isset($this->instances[$abstract]) && ! $needsContextualBuild) {
                  return $this->instances[$abstract];
              }
      
              $this->with[] = $parameters;
      
              $concrete = $this->getConcrete($abstract);
      
      		# 实例化
              if ($this->isBuildable($concrete, $abstract)) {
                  $object = $this->build($concrete);
              } else {
                  $object = $this->make($concrete);
              }
      
      		# 类似于装饰器模式，为某个类添加装饰器，返回装饰后的类，怎么用请看下面附言
              foreach ($this->getExtenders($abstract) as $extender) {
                  $object = $extender($object, $this);
              }
      
      		# 判断是否是单例
              if ($this->isShared($abstract) && ! $needsContextualBuild) {
                  $this->instances[$abstract] = $object;
              }
      
              $this->fireResolvingCallbacks($abstract, $object);
      
      		# 走到这里说明实例化完成，所以把该实例化的抽象类加到 $resolved 属性中
              $this->resolved[$abstract] = true;
      
              array_pop($this->with);
      
              return $object;
          }
      ```

      

      ```php
      public function build($concrete)
      {
          // If the concrete type is actually a Closure, we will just execute it and
          // hand back the results of the functions, which allows functions to be
          // used as resolvers for more fine-tuned resolution of these objects.
          if ($concrete instanceof Closure) {
              return $concrete($this, $this->getLastParameterOverride());
          }
      
          $reflector = new ReflectionClass($concrete);
      
          // If the type is not instantiable, the developer is attempting to resolve
          // an abstract type such as an Interface of Abstract Class and there is
          // no binding registered for the abstractions so we need to bail out.
          if (! $reflector->isInstantiable()) {
              return $this->notInstantiable($concrete);
          }
      
          $this->buildStack[] = $concrete;
      
          $constructor = $reflector->getConstructor();
      
          // If there are no constructors, that means there are no dependencies then
          // we can just resolve the instances of the objects right away, without
          // resolving any other types or dependencies out of these containers.
          if (is_null($constructor)) {
              array_pop($this->buildStack);
      
              return new $concrete;
          }
      
          $dependencies = $constructor->getParameters();
      
          # 会解析参数，如果之前应用绑定过那么拿到的是绑定过得那个参数，单例
          $instances = $this->resolveDependencies(
              $dependencies
          );
      
          array_pop($this->buildStack);
      
          return $reflector->newInstanceArgs($instances);
      }
      ```

      反射类知识

      ```php
      <?php
      namespace Duc;
      
      class B
      {
      }
      
      class A
      {
      
          function __construct(B $duc, $router)
          {
      
          }
      }
      
      $c = (new ReflectionClass(A::class))->getConstructor()->getParameters();
      foreach ($c as $key) {
          var_dump($key->getClass()->name); // Duc\B
      }
      
      # 第二个报错，所以源码用来 try catch
      
      # 源码
      protected function resolveClass(ReflectionParameter $parameter)
      {
          try {
              return $this->make($parameter->getClass()->name);
          }
      
      
          catch (BindingResolutionException $e) {
              if ($parameter->isOptional()) {
                  return $parameter->getDefaultValue();
              }
      
              throw $e;
          }
      }
      
      ```

   2. 内核构造函数

      1. ```php
         public function __construct(Application $app, Router $router)
         {
             $this->app = $app;
             $this->router = $router;
         
             $router->middlewarePriority = $this->middlewarePriority;
         	
             # 保存系统中间件
             foreach ($this->middlewareGroups as $key => $middleware) {
                 $router->middlewareGroup($key, $middleware);
             }
         
             # 添加中间件别名
             foreach ($this->routeMiddleware as $key => $middleware) {
                 $router->aliasMiddleware($key, $middleware);
             }
         }
         ```

         内核具体如何实例化请看文章《Laravel 源码解析之 Kernel 具体生成》

4. 最后执行

   1. ```php
      $response = $kernel->handle(
          $request = Illuminate\Http\Request::capture()
      );
      ```

      ```php
      public function handle($request)
      {
          try {
              $request->enableHttpMethodParameterOverride();
      
              $response = $this->sendRequestThroughRouter($request);
          } catch (Exception $e) {
              $this->reportException($e);
      
              $response = $this->renderException($request, $e);
          } catch (Throwable $e) {
              $this->reportException($e = new FatalThrowableError($e));
      
              $response = $this->renderException($request, $e);
          }
      
          $this->app['events']->dispatch(
              new Events\RequestHandled($request, $response)
          );
      
          return $response;
      }
      ```

      ```php
      protected function sendRequestThroughRouter($request)
      {
          $this->app->instance('request', $request);
      
          Facade::clearResolvedInstance('request');
      
          # 到这一刻系统才算启动成功
          $this->bootstrap();
      
          return (new Pipeline($this->app))
              ->send($request)
              # $this->app->shouldSkipMiddleware() 在测试时用来跳过中间件的
              ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
              ->then($this->dispatchToRouter());
      }
      ```

      其中bootstrap方法：

      加载系统环境变量，加载配置文件，异常处理，注册facads，注册app.php里面的系统的providers，注意这里最后一个 `BootProviders`，会将启动属性改为 true，`$this->booted = true;`，**并且会调用所有已经注册的providers中的boot方法**

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

      

## 附言

### 关于 extend 的使用

```php
# 我这里给 config 加了一层扩展，输出一句 'use extend'。
# 这个方法仅限于扩展系统内部的服务类，类似于代理模式，或者装饰模式，代理可能更加好点
Route::get('/', function () {
    app()->extend('config', function ($config, $app) {
        echo 'use extend';

       return $config;
    });
    
    dd(config('app.name'));
}
           
# 输出：
"Laravel"
use extend
```