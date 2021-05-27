---
title: laravel 源码解析之 View、blade
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - laravel
tags:
 - 源码解析
 - laravel
 - View
 - blade
publish: true
---

> 写了很久的api，最近上手写起页面，很是陌生，blade模板都不知道怎么用了，所以现在来啃下源码！

## 起步

所有的服务注册都从 `ServiceProvider` 起步，所以先来看下 `ViewServiceProvider`

```php
# 起步都是绑定单例
public function register()
{
  $this->registerFactory();

  $this->registerViewFinder();

  $this->registerEngineResolver();
}


public function registerFactory()
{
  $this->app->singleton('view', function ($app) {
    // Next we need to grab the engine resolver instance that will be used by the
    // environment. The resolver will be used by an environment to get each of
    // the various engine implementations such as plain PHP or Blade engine.
    $resolver = $app['view.engine.resolver'];

    $finder = $app['view.finder'];

    $factory = $this->createFactory($resolver, $finder, $app['events']);

    // We will also set the container instance on this view environment since the
    // view composers may be classes registered in the container, which allows
    // for great testable, flexible composers for the application developer.
    $factory->setContainer($app);

    # 所以这里看出所有的view视图都有一个 $app 变量, 这里在每次调用view时，都回去同步app，保持了app最新
    $factory->share('app', $app);

    return $factory;
  });
}

# 下面两个都是为上面的 view factory 做准备，注入了相应的解析器，意味着用户可以随意代替，👍解耦！
public function registerViewFinder()
{
  $this->app->bind('view.finder', function ($app) {
    return new FileViewFinder($app['files'], $app['config']['view.paths']);
  });
}

public function registerEngineResolver()
{
  $this->app->singleton('view.engine.resolver', function () {
    $resolver = new EngineResolver;

    // Next, we will register the various view engines with the resolver so that the
    // environment will resolve the engines needed for various views based on the
    // extension of view file. We call a method for each of the view's engines.
    foreach (['file', 'php', 'blade'] as $engine) {
      $this->{'register'.ucfirst($engine).'Engine'}($resolver);
    }

    return $resolver;
  });
}
```

>  没有 boot 方法说明启动时不需要做什么事情



## view方法

> 一般在控制器中都写这个就可以访问页面了，那么其中的原理是什么呢？

```php
view('welcome');

# 首先在bootstrap 阶段所有的视图都注册完毕
👇
# 先初始化依赖
$resolver = $app['view.engine.resolver'];
// 这里注册了三个引擎
foreach (['file', 'php', 'blade'] as $engine) {
  $this->{'register'.ucfirst($engine).'Engine'}($resolver);
}

$finder = $app['view.finder'];

# 接着在别的 ServiceProvider 中注册视图，比如 NotificationServiceProvider
$this->loadViewsFrom(__DIR__.'/resources/views', 'notifications');
```



## 加载视图

```php
protected function loadViewsFrom($path, $namespace)
{
  if (is_array($this->app->config['view']['paths'])) {
    foreach ($this->app->config['view']['paths'] as $viewPath) {
      if (is_dir($appPath = $viewPath.'/vendor/'.$namespace)) {
        $this->app['view']->addNamespace($namespace, $appPath);
      }
    }
  }

  $this->app['view']->addNamespace($namespace, $path);
}

public function addNamespace($namespace, $hints)
{
  $hints = (array) $hints;

  if (isset($this->hints[$namespace])) {
    $hints = array_merge($this->hints[$namespace], $hints);
  }

  $this->hints[$namespace] = $hints;
}
```

1. 查看工作目录的 resource/views 下有没有 vendor 目录，并检查其中是否存在和要注册视图一样的命名空间，有的话就注册
2. 把服务提供商自己的视图加进去



例如：

存在 vendor/notifications/views/email.blade.php，那么 hits 就是

```php
$hits = [
  "/Users/congcong/duc/app60/resources/views/vendor/notifications",
 "/Users/congcong/duc/app60/vendor/laravel/framework/src/Illuminate/Notifications/resources/views"
];
```



## 启动完成后

```php
view('welcome');

public function make($view, $data = [], $mergeData = [])
{
  $path = $this->finder->find(
    $view = $this->normalizeName($view)
  );

  # 第二个参数由下面可知只要实现了 Arrayable 的方法都可以传入
  # 由此可知，view 的第三个参数不会被解析原木原样传入
  # data 是一个数组，第二个参数的key、会覆盖第三个的可以，注意merge，数字key会被重新索引
  $data = array_merge($mergeData, $this->parseData($data));
  

  return tap($this->viewInstance($view, $path, $data), function ($view) {
    $this->callCreator($view);
  });
}

# 如果带命名空间 比如 admin::welcome, 那么返回 admin::welcome , 再如 admin::user/welcome返回 admin::user.welcome
public static function normalize($name)
{
  $delimiter = ViewFinderInterface::HINT_PATH_DELIMITER;

  if (strpos($name, $delimiter) === false) {
    return str_replace('/', '.', $name);
  }

  [$namespace, $name] = explode($delimiter, $name);

  return $namespace.$delimiter.str_replace('/', '.', $name);
}

# 返回视图路径
public function find($name)
{
  if (isset($this->views[$name])) {
    return $this->views[$name];
  }

  if ($this->hasHintInformation($name = trim($name))) {
    return $this->views[$name] = $this->findNamespacedView($name);
  }

  return $this->views[$name] = $this->findInPaths($name, $this->paths);
}

protected function findInPaths($name, $paths)
{
  foreach ((array) $paths as $path) {
    foreach ($this->getPossibleViewFiles($name) as $file) {
      if ($this->files->exists($viewPath = $path.'/'.$file)) {
        return $viewPath;
      }
    }
  }

  throw new InvalidArgumentException("View [{$name}] not found.");
}

protected function getPossibleViewFiles($name)
{
  return array_map(function ($extension) use ($name) {
    return str_replace('.', '/', $name).'.'.$extension;
  }, $this->extensions);
}

# 可知，能匹配到这四个结尾的文件
protected $extensions = ['blade.php', 'php', 'css', 'html'];


# 由此可知只要实现了 Arrayable 的方法都可以传入
protected function parseData($data)
{
  return $data instanceof Arrayable ? $data->toArray() : $data;
}
```

```php
return tap($this->viewInstance($view, $path, $data), function ($view) {
  $this->callCreator($view);
});

# 选用 blade.php 引擎进行渲染
protected function viewInstance($view, $path, $data)
{
  return new View($this, $this->getEngineFromPath($path), $view, $path, $data);
}

# 意思是通过事件触发的渲染
public function callCreator(ViewContract $view)
{
  $this->events->dispatch('creating: '.$view->name(), [$view]);
}
```

至此控制器返回了 View  实例，然后交给router作返回处理 

```php
public function prepareResponse($request, $response)
{
    return static::toResponse($request, $response);
}
```

```php
public function setContent($content)
{
  $this->original = $content;

  // If the content is "JSONable" we will set the appropriate header and convert
  // the content to JSON. This is useful when returning something like models
  // from routes that will be automatically transformed to their JSON form.
  if ($this->shouldBeJson($content)) {
    $this->header('Content-Type', 'application/json');

    $content = $this->morphToJson($content);
  }

  # 这里渲染 view 视图
  elseif ($content instanceof Renderable) {
    $content = $content->render();
  }

  parent::setContent($content);

  return $this;
}
```



```php
/**
     * Get the contents of the view instance.
     *
     * @return string
     */
protected function renderContents()
{
  // We will keep track of the amount of views being rendered so we can flush
  // the section after the complete rendering operation is done. This will
  // clear out the sections for any separate views that may be rendered.
  $this->factory->incrementRender();

  # 触发事件 "composing: welcome" event
  $this->factory->callComposer($this);

  $contents = $this->getContents();

  // Once we've finished rendering the view, we'll decrement the render count
  // so that each sections get flushed out next time a view is created and
  // no old sections are staying around in the memory of an environment.
  $this->factory->decrementRender();

  return $contents;
}

public function gatherData()
{
  $data = array_merge($this->factory->getShared(), $this->data);
# 获取注入的变量默认已经注入 __env, errors, app 三个
  foreach ($data as $key => $value) {
    if ($value instanceof Renderable) {
      # 这里意味着还能注入视图(Renderable)
      $data[$key] = $value->render();
    }
  }

  return $data;
}


public function collectViewData($path, array $data): void
{
  $this->lastCompiledData[] = [
    'path' => $path,
    'compiledPath' => $this->getCompiledPath($path),
    'data' => $this->filterViewData($data),
  ];
}

 protected function getCompiledPath($path): string
 {
   if ($this instanceof CompilerEngine) {
     return $this->getCompiler()->getCompiledPath($path);
   }

   return $path;
 }

# 用 sha1 哈希了路径
# 这一步拿了缓存路径
public function getCompiledPath($path)
{
  return $this->cachePath.'/'.sha1($path).'.php';
}

```

```php
public function get($path, array $data = [])
{
  $this->lastCompiled[] = $path;

 # 先查看是否缓存了视图以及缓存的视图是否被更改过，如果是，则重新编译
  if ($this->compiler->isExpired($path)) {
    $this->compiler->compile($path);
  }

  $compiled = $this->compiler->getCompiledPath($path);

  // Once we have the path to the compiled file, we will evaluate the paths with
  // typical PHP just like any other templates. We also keep a stack of views
  // which have been rendered for right exception messages to be generated.
  $results = $this->evaluatePath($compiled, $data);

  array_pop($this->lastCompiled);

  return $results;
}

# 判断是否过期，用原文件的最后修改时间和缓存文件的最后修改时间作比较
public function isExpired($path)
{
  $compiled = $this->getCompiledPath($path);

  // If the compiled file doesn't exist we will indicate that the view is expired
  // so that it can be re-compiled. Else, we will verify the last modification
  // of the views is less than the modification times of the compiled views.
  if (! $this->files->exists($compiled)) {
    return true;
  }

  return $this->files->lastModified($path) >=
    $this->files->lastModified($compiled);
}


# 直接把blade include 进来了
 protected function evaluatePath($__path, $__data)
 {
   $obLevel = ob_get_level();

   ob_start();

   extract($__data, EXTR_SKIP);

   // We'll evaluate the contents of the view inside a try/catch block so we can
   // flush out any stray output that might get out before an error occurs or
   // an exception is thrown. This prevents any partial views from leaking.
   try {
     include $__path;
   } catch (Exception $e) {
     $this->handleViewException($e, $obLevel);
   } catch (Throwable $e) {
     $this->handleViewException(new FatalThrowableError($e), $obLevel);
   }

   return ltrim(ob_get_clean());
 }
```

```php
public function compile($path = null)
{
  if ($path) {
    $this->setPath($path);
  }

  if (! is_null($this->cachePath)) {
    $contents = $this->compileString(
      # 这里用file_get_contents把整个blade读进来了
      $this->files->get($this->getPath())
    );

    if (! empty($this->getPath())) {
      $tokens = $this->getOpenAndClosingPhpTokens($contents);

      // If the tokens we retrieved from the compiled contents have at least
      // one opening tag and if that last token isn't the closing tag, we
      // need to close the statement before adding the path at the end.
      if ($tokens->isNotEmpty() && $tokens->last() !== T_CLOSE_TAG) {
        $contents .= ' ?>';
      }

      $contents .= "<?php /**PATH {$this->getPath()} ENDPATH**/ ?>";
    }

    $this->files->put(
      $this->getCompiledPath($this->getPath()), $contents
    );
  }
}

# 编译读进来的视图字符串
public function compileString($value)
{
  if (strpos($value, '@verbatim') !== false) {
    $value = $this->storeVerbatimBlocks($value);
  }

  $this->footer = [];

  if (strpos($value, '@php') !== false) {
    $value = $this->storePhpBlocks($value);
  }

  $result = '';

# 处理了
# 'Comments',  注释 {{-- xxxx --}} 直接用 '' 替换
  'Extensions',
# 'Statements', 处理了 @php @foreach 等等的标签，
// 比如 @if(Route::has('login')) 转换成
// <?php if(Route::has('login')): ?>
  'Echos',
# 的标签
  foreach (token_get_all($value) as $token) {
    $result .= is_array($token) ? $this->parseToken($token) : $token;
  }

  if (! empty($this->rawBlocks)) {
    $result = $this->restoreRawContent($result);
  }

  // If there are any footer lines that need to get added to a template we will
  // add them here at the end of the template. This gets used mainly for the
  // template inheritance via the extends keyword that should be appended.
  if (count($this->footer) > 0) {
    $result = $this->addFooters($result);
  }

  return $result;
}

```

## 标签解释器在这里👇

```php
Illuminate\View\Compilers\Concerns\CompilesConditionals;
```

编译完之后的模板用 file_put_contents 写入到缓存目录，所以这就表明每次访问时都会生成缓存文件，为什么没有察觉到是因为缓存文件做了 `isExpired()` 判断



## 课外

> [token_get_all 将提供的源码按 PHP 标记进行分割 ](https://www.php.net/manual/zh/function.token-get-all.php)

```php
<?php
$tokens = token_get_all('<?php echo; ?>'); /* => array(
                                                  array(T_OPEN_TAG, '<?php'), 
                                                  array(T_ECHO, 'echo'),
                                                  ';',
                                                  array(T_CLOSE_TAG, '?>') ); */
```



暂时读到这里吧，基本的核心逻辑都搞懂了🧐



-----------



## laravel 7 发布之后多了一个 blade-x

核心编译逻辑

```php
public function compileTags(string $value)
{
  $value = $this->compileSelfClosingTags($value);
  $value = $this->compileOpeningTags($value);
  $value = $this->compileClosingTags($value);

  return $value;
}
```

```php
// 编译 x-tag 的地方
$value = $this->compileOpeningTags($value);
```

这里解释了为什么 可以用 make:component 生成的 class 来替换view

```php
protected function componentClass(string $component)
{
  if (isset($this->aliases[$component])) {
    return $this->aliases[$component];
  }

  // 这里先判断 make:component class 文件是否存在
  if (class_exists($class = $this->guessClassName($component))) {
    return $class;
  }

  // 不存在就去 解析 components.{$component}
  if (Container::getInstance()->make(Factory::class)
      ->exists($view = "components.{$component}")) {
    return $view;
  }

  throw new InvalidArgumentException(
    "Unable to locate a class or view for component [{$component}]."
  );
}
```

编译前

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>test</title>
</head>
<body>
    <p>test</p>
    <x-tag></x-tag>
</body>
</html>
```

编译后把 `<x-tag>` 替换成了 `@component` 语法

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>test</title>
</head>
<body>
    <p>test</p>
     @component('Illuminate\View\AnonymousComponent', ['view' => 'components.tag','data' => []])
<?php $component->withAttributes([]); ?></x-tag>
</body>
</html>
```




