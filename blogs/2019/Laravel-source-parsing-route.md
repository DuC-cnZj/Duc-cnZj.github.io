---
title: laravel 源码解析之 路由
date: "2019-04-02 12:40:23"
sidebar: true
categories:
 - 技术
tags:
 - 源码解析
 - 路由
 - laravel
publish: true
---


> `Illuminate\Routing` 和路由相关的类有一堆

我们就从 `RoutingServiceProvider` 和  `RouteServiceProvider` 说起吧，`RoutingServiceProvider` 在框架启动阶段注册了它，作为核心的三大服务之一，它进行了以下绑定

```php
public function register()
{
    $this->registerRouter();
    $this->registerUrlGenerator();
    $this->registerRedirector();
    $this->registerPsrRequest();
    $this->registerPsrResponse();
    $this->registerResponseFactory();
    $this->registerControllerDispatcher();
}

# 做了以下绑定
# (singleton) router => Illuminate\Routing\Router
# (singleton) url => Illuminate\Routing\UrlGenerator
# (singleton) redirect => Illuminate\Routing\Redirector
# (bind) ServerRequestInterface::class => (new DiactorosFactory)->createRequest($app->make('request'));
# (bind) ResponseInterface::class => Zend\Diactoros\Response
# (singleton) ResponseFactoryContract::class => Illuminate\Routing\ResponseFactory
# (singleton) ControllerDispatcherContract::class => Illuminate\Routing\ControllerDispatcher
```

然后在 `bootstrap` 阶段，框架调用了注册服务的 `register` 和 `boot` 方法

```php
# RouteServiceProvider
public function boot()
{
    //

    parent::boot();
}

# 父类中的 boot
public function boot()
{
    # 为 url 类设置了 rootNamespace 为 'App\Http\Controllers'
    $this->setRootControllerNamespace();

    if ($this->routesAreCached()) {
        # 调用缓存
        $this->loadCachedRoutes();
    } else {
        # 加载路由
        $this->loadRoutes();

        $this->app->booted(function () {
            $this->app['router']->getRoutes()->refreshNameLookups();
            $this->app['router']->getRoutes()->refreshActionLookups();
        });
    }
}

# 使用 app 的 call 方法(这个方法有点复杂)，总之调用了自身的map
protected function loadRoutes()
{
    if (method_exists($this, 'map')) {
        $this->app->call([$this, 'map']);
    }
}

public function map()
{
    # 加载 api 路由
    $this->mapApiRoutes();

    # 加载 web 路由
    $this->mapWebRoutes();

    //
}
```

## 路由加载过程

```php
protected function mapApiRoutes()
{
    Route::prefix('api')
        ->middleware('api')
        ->namespace($this->namespace)
        ->group(base_path('routes/api.php'));
    # 注意，如果用户自己也在路由中定义了 prefix 属性，那么将替换 '/api' 前缀
}
```

首先 调用 Route Facade，也就是 `\Illuminate\Routing\Router` 类，因为类中没有，所以走 `__call`，为什么是 `__call` 而不是 `__callStatic`，facade 底层通过 app 实例化类后 `$instance->method()`这样调用的。

```php
public function __call($method, $parameters)
{
    # 我们没有定义宏，所以这里是 false
    if (static::hasMacro($method)) {
        return $this->macroCall($method, $parameters);
    }

    if ($method === 'middleware') {
        return (new RouteRegistrar($this))->attribute($method, is_array($parameters[0]) ? $parameters[0] : $parameters);
    }

    return (new RouteRegistrar($this))->attribute($method, $parameters[0]);
}
```

RouteRegistrar 掌管路由注册

```php
# key => 'prefix', value => 'api'
public function attribute($key, $value)
{
    # allowedAttributes 属性的定义意味着你可以链式调用该属性中的所有方法
    # Route::[as|domain|middleware|name|namespace|prefix|where]()
    if (! in_array($key, $this->allowedAttributes)) {
        throw new InvalidArgumentException("Attribute [{$key}] does not exist.");
    }

    # 这一步设置了 RouteRegistrar 的 $attributes 属性
    # aliases：意味着你的 name 方法最后对应的属性是 as
    # example:
    # $attributes = [
    # 'as' => '', 'domain' => '', 'middleware' => '', 'name' => '', 'namespace' => '', 'prefix' => '', 'where' => '',
    # ]
    $this->attributes[Arr::get($this->aliases, $key, $key)] = $value;

    # 返回本身，意味着可以链式调用
    return $this;
}
```

```php
protected $aliases = [
    'name' => 'as',
];
protected $allowedAttributes = [
    'as', 'domain', 'middleware', 'name', 'namespace', 'prefix', 'where',
];
```

调用`middleware`方法时，它另做了处理，传入了数组，因为中间件本身应该是一个数组，而其他是字符串

```php
if ($method === 'middleware') {
    return (new RouteRegistrar($this))->attribute($method, is_array($parameters[0]) ? $parameters[0] : $parameters);
}
```

链式的最后是 group 方法

```php
public function group($callback)
{
    # 把上面的属性数组传入 router 类
    $this->router->group($this->attributes, $callback);
}

public function group(array $attributes, $routes)
{
    # 做暂存用，把要处理的数据压栈
    $this->updateGroupStack($attributes);

    $this->loadRoutes($routes);

    # 处理完的数据出栈
    array_pop($this->groupStack);
}

protected function updateGroupStack(array $attributes)
{
    if (! empty($this->groupStack)) {
        $attributes = $this->mergeWithLastGroup($attributes);
    }

    $this->groupStack[] = $attributes;
}
```

加载用户定义的路由

```php
protected function loadRoutes($routes)
{
    if ($routes instanceof Closure) {
        $routes($this);
    } else {
        # 上面传入 base_path('routes/api.php') 所以走这里
        (new RouteFileRegistrar($this))->register($routes);
    }
}
```

```php
public function register($routes)
{
    $router = $this->router;

    # 包含了用户自定义的路由文件，可以看到，你在 api.php 中不仅可以使用 Route Facade，也可以直接使用 $router，而使用任何一个都拿到的是同一个实例，因为最后解析到的都是 Router 类，而这个类的单例 singleton
    require $routes;
}
```

```php
Route::middleware('auth:api')->get('/user', function (Request $request) {
    return $request->user();
});

# 和下面的写法相等

$router->middleware('auth:api')->get('/user', function (Request $request) {
    return $request->user();
});
```

⁉️注意，这里的 `$router->middleware('auth:api')->get(...)` 调用的是 RouteRegistrar 类的 get 方法，因为 middleware 返回的是 RouteRegistrar 实例

```php
# RouteRegistrar.php
public function __call($method, $parameters)
{
    if (in_array($method, $this->passthru)) {
        return $this->registerRoute($method, ...$parameters);
    }

    if (in_array($method, $this->allowedAttributes)) {
        if ($method === 'middleware') {
            return $this->attribute($method, is_array($parameters[0]) ? $parameters[0] : $parameters);
        }

        return $this->attribute($method, $parameters[0]);
    }

    throw new BadMethodCallException(sprintf(
        'Method %s::%s does not exist.', static::class, $method
    ));
}

protected $passthru = [
    'get', 'post', 'put', 'patch', 'delete', 'options', 'any',
];

protected function registerRoute($method, $uri, $action = null)
{
    if (! is_array($action)) {
        $action = array_merge($this->attributes, $action ? ['uses' => $action] : []);
    }

    return $this->router->{$method}($uri, $this->compileAction($action));
}

# 把 api.php 中的闭包格式化成 ['uses' => Closure] 并且合并了对应的 router 中的其他属性 prefix,middleware,...
protected function compileAction($action)
{
    if (is_null($action)) {
        return $this->attributes;
    }

    if (is_string($action) || $action instanceof Closure) {
        $action = ['uses' => $action];
    }

    return array_merge($this->attributes, $action);
}

# Router.php
public function get($uri, $action = null)
{
    return $this->addRoute(['GET', 'HEAD'], $uri, $action);
}

public function addRoute($methods, $uri, $action)
{
    # $this->routes => \Illuminate\Routing\RouteCollection
    return $this->routes->add($this->createRoute($methods, $uri, $action));
}
```

在使用 `RouteCollection`之前，先 `createRoute`

```php
protected function createRoute($methods, $uri, $action)
{
	# $method => get, $uri => /user, $action => Closure
    # 这里判断了是否使用了 'XxxController@method' 的方式，显然不是
    if ($this->actionReferencesController($action)) {
        $action = $this->convertToControllerAction($action);
    }

    # Illuminate\Routing\Route 实例，上面的 Route 说的是 facade ，这个不是
    $route = $this->newRoute(
        $methods, $this->prefix($uri), $action
    );


    # 设置其他的全局属性，上面不是把一些 middlrware、prefix 压栈了吗，把那些属性和此路由本身的属性 merge
    if ($this->hasGroupStack()) {
        $this->mergeGroupAttributesIntoRoute($route);
    }

    # 合并 where 查询，这里存的是数组，那么意味着，可以匹配多个规则
    $this->addWhereClausesToRoute($route);

    return $route;
}


protected function newRoute($methods, $uri, $action)
{
    return (new Route($methods, $uri, $action))
        ->setRouter($this)
        ->setContainer($this->container);
}
```

`Route` 类

```php
public function __construct($methods, $uri, $action)
{
    $this->uri = $uri; # /user
    $this->methods = (array) $methods; // ['GET', 'HEAD']
    $this->action = $this->parseAction($action);

    if (in_array('GET', $this->methods) && ! in_array('HEAD', $this->methods)) {
        $this->methods[] = 'HEAD';
    }

    if (isset($this->action['prefix'])) {
        $this->prefix($this->action['prefix']);
    }
}

# 主要是为了解析出 uses 属性，还有只写控制器不写方法的情况需要__invoke 出来 'HomeController' => ['uses' => 'HomeController@__invoke']
protected function parseAction($action)
{
    return RouteAction::parse($this->uri, $action);
}
```

调用 `RouteCollection->add`方法，把路由添加到 `Router`的`routes(\Illuminate\Routing\RouteCollection 的实例)` 的 `routes` 和 `allRoutes`属性中

```php
#routes: RouteCollection {#30 ▼
      #routes: array:3 [▼
        "GET" => array:10 [▼
          "laravel-websockets" => Route {#126 ▶}
          "laravel-websockets/api/{appId}/statistics" => Route {#127 ▶}
          "broadcasting/auth" => Route {#151 ▶}
          "api/user" => Route {#165 ▶}
          "/" => Route {#167 ▶}
          "login" => Route {#168 ▶}
          "register" => Route {#171 ▶}
          "password/reset" => Route {#173 ▶}
          "password/reset/{token}" => Route {#175 ▶}
          "duc/duc" => Route {#178 ▶}
        ]
        "HEAD" => array:10 [▼
          "laravel-websockets" => Route {#126 ▶}
          "laravel-websockets/api/{appId}/statistics" => Route {#127 ▶}
          "broadcasting/auth" => Route {#151 ▶}
          "api/user" => Route {#165 ▶}
          "/" => Route {#167 ▶}
          "login" => Route {#168 ▶}
          "register" => Route {#171 ▶}
          "password/reset" => Route {#173 ▶}
          "password/reset/{token}" => Route {#175 ▶}
          "duc/duc" => Route {#178 ▶}
        ]
        "POST" => array:9 [▶]
      ]
      #allRoutes: array:18 [▼
        "HEADlaravel-websockets" => Route {#126 ▶}
        "HEADlaravel-websockets/api/{appId}/statistics" => Route {#127 ▶}
        "POSTlaravel-websockets/auth" => Route {#128 ▶}
        "POSTlaravel-websockets/event" => Route {#129 ▶}
        "POSTlaravel-websockets/statistics" => Route {#130 ▶}
        "HEADbroadcasting/auth" => Route {#151 ▶}
        "HEADapi/user" => Route {#165 ▶}
        "HEAD/" => Route {#167 ▶}
        "HEADlogin" => Route {#168 ▶}
        "POSTlogin" => Route {#169 ▶}
        "POSTlogout" => Route {#170 ▶}
        "HEADregister" => Route {#171 ▶}
        "POSTregister" => Route {#172 ▶}
        "HEADpassword/reset" => Route {#173 ▶}
        "POSTpassword/email" => Route {#174 ▶}
        "HEADpassword/reset/{token}" => Route {#175 ▶}
        "POSTpassword/reset" => Route {#176 ▶}
        "HEADduc/duc" => Route {#178 ▶}
      ]
      #nameList: array:1 [▼
        "duc" => Route {#178 ▶}
      ]
      #actionList: array:15 [▼
        "BeyondCode\LaravelWebSockets\Dashboard\Http\Controllers\ShowDashboard" => Route {#126 ▶}
        "BeyondCode\LaravelWebSockets\Dashboard\Http\Controllers\DashboardApiController@getStatistics" => Route {#127 ▶}
        "BeyondCode\LaravelWebSockets\Dashboard\Http\Controllers\AuthenticateDashboard" => Route {#128 ▶}
        "BeyondCode\LaravelWebSockets\Dashboard\Http\Controllers\SendMessage" => Route {#129 ▶}
        "BeyondCode\LaravelWebSockets\Statistics\Http\Controllers\WebSocketStatisticsEntriesController@store" => Route {#130 ▶}
        "Illuminate\Broadcasting\BroadcastController@authenticate" => Route {#151 ▶}
        "App\Http\Controllers\Auth\LoginController@showLoginForm" => Route {#168 ▶}
        "App\Http\Controllers\Auth\LoginController@login" => Route {#169 ▶}
        "App\Http\Controllers\Auth\LoginController@logout" => Route {#170 ▶}
        "App\Http\Controllers\Auth\RegisterController@showRegistrationForm" => Route {#171 ▶}
        "App\Http\Controllers\Auth\RegisterController@register" => Route {#172 ▶}
        "App\Http\Controllers\Auth\ForgotPasswordController@showLinkRequestForm" => Route {#173 ▶}
        "App\Http\Controllers\Auth\ForgotPasswordController@sendResetLinkEmail" => Route {#174 ▶}
        "App\Http\Controllers\Auth\ResetPasswordController@showResetForm" => Route {#175 ▶}
        "App\Http\Controllers\Auth\ResetPasswordController@reset" => Route {#176 ▶}
      ]
    }
```

之后的每条路由都是这样加入，就不讲了。

接下来继续看 `boot` 方法 `$this->loadRoutes();` 之后的事情

```php
public function boot()
{
    $this->setRootControllerNamespace();

    if ($this->routesAreCached()) {
        $this->loadCachedRoutes();
    } else {
        $this->loadRoutes();

        # 设置启动后回调，调用时间是在 启动最后一步 调用 \Illuminate\Foundation\Bootstrap\BootProviders::class 时，主要是为了防止路由注册后用户又对路由做的改动
        $this->app->booted(function () {
            # 刷新 router 类的 routes 实例属性的nameList 属性
            $this->app['router']->getRoutes()->refreshNameLookups();
            # 刷新 router 类的 routes 实例属性的nameList 属性
            $this->app['router']->getRoutes()->refreshActionLookups();
        });
    }
}


public function booted($callback)
{
    $this->bootedCallbacks[] = $callback;

    if ($this->isBooted()) {
        $this->fireAppCallbacks([$callback]);
    }
}
```

| class                        | 用途                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| Route                        | 每个子路由的实例                                             |
| RouteAction                  | 把 'HomeController@index' 解析成闭包，即 ['uses' => \Closure] |
| RouteBinding                 |                                                              |
| RouteCollection              | Router 中的 routes 实例                                      |
| RouteCompiler                |                                                              |
| RouteDependencyResolverTrait |                                                              |
| RouteFileRegistrar           |                                                              |
| RouteGroup                   |                                                              |
| RouteParameterBinder         |                                                              |
| Router                       | app('router') 实例                                           |
| RouteRegistrar               | 路由注册器，掌管所有的路由注册处理                           |
| RouteSignatureParameters     |                                                              |
| RouteUrlGenerator            |                                                              |
| RoutingServiceProvider       |                                                              |
| SortedMiddleware             |                                                              |
| UrlGenerator                 |                                                              |
| ...                          |                                                              |

到此为止，所有的路由注册完成，至于怎么命中对应的路由，请看下回分析。😁




# laravel 7 新增自定义 slug

```php
Route::get('/users/{user}/names/{name:name}', function (User $user, \App\Name $name) {
    dd($user, $name);
});
```

原理是在路由解析时 Route.php 中新增一个 bindingFields 字段，保存了 ['name' => 'name'] ,也就是slug值

```php
public function __construct($methods, $uri, $action)
{
  $this->uri = $this->parseUri($uri);
  $this->methods = (array) $methods;
  $this->action = $this->parseAction($action);

  if (in_array('GET', $this->methods) && ! in_array('HEAD', $this->methods)) {
    $this->methods[] = 'HEAD';
  }
```

然后在 SubstituteBindings 中间件中解析了，

```php
public static function resolveForRoute($container, $route)
{
  $parameters = $route->parameters();

  foreach ($route->signatureParameters(UrlRoutable::class) as $parameter) {
    if (! $parameterName = static::getParameterName($parameter->name, $parameters)) {
      continue;
    }

    $parameterValue = $parameters[$parameterName];

    if ($parameterValue instanceof UrlRoutable) {
      continue;
    }

    $instance = $container->make($parameter->getClass()->name);

    $parent = $route->parentOfParameter($parameterName);
		// 看到这里 bindingFieldFor 方法就是获取对应的slug
    if ($parent instanceof UrlRoutable && $route->bindingFieldFor($parameterName)) {
      if (! $model = $parent->resolveChildRouteBinding(
        $parameterName, $parameterValue, $route->bindingFieldFor($parameterName)
      )) {
        throw (new ModelNotFoundException)->setModel(get_class($instance), [$parameterValue]);
      }
    } elseif (! $model = $instance->resolveRouteBinding($parameterValue, $route->bindingFieldFor($parameterName))) {
      throw (new ModelNotFoundException)->setModel(get_class($instance), [$parameterValue]);
    }

    $route->setParameter($parameterName, $model);
  }
}
```

然后在 传入第三个参数 name

```php
public function resolveRouteBinding($value, $field = null)
{
  return $this->where($field ?? $this->getRouteKeyName(), $value)->first();
}
```

这时候如果设置了 slug 那么就是 

```php
$this->where('name', $value)->first();
```

如果没有那么就是 getRouteKeyName(): 一般也就是 id，不过可以自己在model定义



## 作用域绑定

比如有这样的数据

```json
[
    {
        "id":1,
        "user_id":"1",
        "name":"duc",
        "created_at":"2020-03-07T03:46:43.000000Z",
        "updated_at":"2020-03-07T03:46:43.000000Z",
        "user":{
            "id":1,
            "name":"Carissa Koelpin",
            "email":"kfritsch@example.com",
            "email_verified_at":"2020-03-07T03:46:43.000000Z",
            "created_at":"2020-03-07T03:46:43.000000Z",
            "updated_at":"2020-03-07T03:46:43.000000Z"
        }
    },
    {
        "id":2,
        "user_id":"2",
        "name":"abc",
        "created_at":"2020-03-07T03:46:43.000000Z",
        "updated_at":"2020-03-07T03:46:43.000000Z",
        "user":{
            "id":2,
            "name":"Eulah Koelpin",
            "email":"hheaney@example.net",
            "email_verified_at":"2020-03-07T03:46:43.000000Z",
            "created_at":"2020-03-07T03:46:43.000000Z",
            "updated_at":"2020-03-07T03:46:43.000000Z"
        }
    }
]
```

有这样的路由

```php
Route::get('/users/{user}/names/{name:name}', function (User $user, Name $name) {
    dd($user, $name);
});
```

然后请求 `/users/1/names/abc` 会报错 ModelNotFoundException, 因为 name 的上层不是 user:1, 如何实现的呢？往下看

```php
public static function resolveForRoute($container, $route)
{
  $parameters = $route->parameters();

  foreach ($route->signatureParameters(UrlRoutable::class) as $parameter) {
    if (! $parameterName = static::getParameterName($parameter->name, $parameters)) {
      continue;
    }

    $parameterValue = $parameters[$parameterName];

    if ($parameterValue instanceof UrlRoutable) {
      continue;
    }

    $instance = $container->make($parameter->getClass()->name);

    // 当去解析第二个模型绑定时，框架会去拿第一个绑定的值，如果第一个是模型绑定的也就是模型实例，并且 $route->bindingFieldFor($parameterName) 有值，代表用户设置了 slug 那么会去 resolveChildRouteBinding
    $parent = $route->parentOfParameter($parameterName);

    if ($parent instanceof UrlRoutable && $route->bindingFieldFor($parameterName)) {
      if (! $model = $parent->resolveChildRouteBinding(
        $parameterName, $parameterValue, $route->bindingFieldFor($parameterName)
      )) {
        throw (new ModelNotFoundException)->setModel(get_class($instance), [$parameterValue]);
      }
    } elseif (! $model = $instance->resolveRouteBinding($parameterValue, $route->bindingFieldFor($parameterName))) {
      throw (new ModelNotFoundException)->setModel(get_class($instance), [$parameterValue]);
    }

    $route->setParameter($parameterName, $model);
  }
}
```

resolveChildRouteBinding 会去生成这样的查询 `$user->name()->where('name', $name)->first()`

```php
public function resolveChildRouteBinding($childType, $value, $field)
{
  return $this->{Str::plural($childType)}()->where($field, $value)->first();
}
```

Ok,懂了吧



如果你想使用默认的id slug，又想解析子绑定，那么需要这样写

```php
Route::get('/users/{user}/names/{name:id}', function (User $user, Name $name) {
    dd($user, $name);
});
```

> ‼️注意前提是 `User $user, Name $name` 你的两个路由参数都做了模型绑定，并且后者还使用了 slug 绑定