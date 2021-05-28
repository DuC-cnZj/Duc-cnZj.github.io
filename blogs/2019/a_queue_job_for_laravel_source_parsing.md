---
title: Laravel 源码解析之 队列Job
date: "2019-01-25 14:27:12"
sidebar: true
categories:
 - 技术
tags:
 - laravel
 - 源码解析
 - Job
publish: true
---


## 先从服务注册开始讲起

```php
# 队列服务是延迟加载的，即调用到才去注册加载对应的服务
# 我们先来看 register 
class QueueServiceProvider extends ServiceProvider
{
    /**
     * Indicates if loading of the provider is deferred.
     *
     * @var bool
     */
    protected $defer = true;

    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        $this->registerManager();
        $this->registerConnection();
        $this->registerWorker();
        $this->registerListener();
        $this->registerFailedJobServices();
        $this->registerOpisSecurityKey();
    }
    
    # 1
    protected function registerManager()
    {
        $this->app->singleton('queue', function ($app) {
			# 返回 QueueManager 实例，并且注册基本的队列连接
            return tap(new QueueManager($app), function ($manager) {
                $this->registerConnectors($manager);
            });
        });
    }
    
    public function registerConnectors($manager)
    {
        foreach (['Null', 'Sync', 'Database', 'Redis', 'Beanstalkd', 'Sqs'] as $connector) {
            $this->{"register{$connector}Connector"}($manager);
        }
    }

    # 随便拿个来看看
    # 这一步看出框架事件添加了 QueueManager 里面的 connectors 属性
    protected function registerRedisConnector($manager)
    {
        $manager->addConnector('redis', function () {
            return new RedisConnector($this->app['redis']);
        });
    }
    
    public function addConnector($driver, Closure $resolver)
    {
        $this->connectors[$driver] = $resolver;
    }
    
    # 2
    # 单例绑定 queue.connection 对应响应的队列处理类，如 RedisQueue 实例
    protected function registerConnection()
    {
        $this->app->singleton('queue.connection', function ($app) {
            return $app['queue']->connection();
        });
    }
    
    # 3
    # 注册 worker
    protected function registerWorker()
    {
        $this->app->singleton('queue.worker', function () {
            return new Worker(
                $this->app['queue'], $this->app['events'], $this->app[ExceptionHandler::class]
            );
        });
    }
    
    # 4
    # 注册 listener
    protected function registerListener()
    {
        $this->app->singleton('queue.listener', function () {
            return new Listener($this->app->basePath());
        });
    }
    
    # 5
    # 注册队列失败处理器
    protected function registerFailedJobServices()
    {
        $this->app->singleton('queue.failer', function () {
            $config = $this->app['config']['queue.failed'];

            return isset($config['table'])
                        ? $this->databaseFailedJobProvider($config)
                        : new NullFailedJobProvider;
        });
    }
    
    # 6 安全相关
    protected function registerOpisSecurityKey()
    {
        if (Str::startsWith($key = $this->app['config']->get('app.key'), 'base64:')) {
            $key = base64_decode(substr($key, 7));
        }

        SerializableClosure::setSecretKey($key);
    }
```

注册完之后

我们在控制器中调用

```php
public function index()
{
    $this->dispatch(new DucJob());
}

# 实际上调用了 Illuminate\Foundation\Bus\Dispatcher 的 dispatch 方法
# 这个 Dispatcher 是 Illuminate\Bus\Dispatcher 类的实例
protected function dispatch($job)
{
    return app(Dispatcher::class)->dispatch($job);
}

#  注意这里的 $command 参数其实是我们的 job 类实例
public function dispatch($command)
{
    # queueResolver 在注册的时候绑定过
    # 判断是否实现 ShouldQueue 接口
    # 很显然 true
    if ($this->queueResolver && $this->commandShouldBeQueued($command)) {
        return $this->dispatchToQueue($command);
    }

    return $this->dispatchNow($command);
}

# 绑定 Dispatcher 的时候会把对应的闭包传入给到 queueResolver
# 这里的 QueueFactoryContract 类其实就是 app['queue'] 实例，也就是 QueueManager 实例
$this->app->singleton(Dispatcher::class, function ($app) {
    return new Dispatcher($app, function ($connection = null) use ($app) {
        return $app[QueueFactoryContract::class]->connection($connection);
    });
});


public function dispatchToQueue($command)
{
    # 没有在 job 中设置连接的话，默认是 null
    $connection = $command->connection ?? null;

    # 调用 QueueManager 的 connection 方法, 传入连接 为 null
    $queue = call_user_func($this->queueResolver, $connection);

    if (! $queue instanceof Queue) {
        throw new RuntimeException('Queue resolver did not return a Queue implementation.');
    }

    # 如果用户对应的 job 有 queue 方法那么执行，使用场景？？
    if (method_exists($command, 'queue')) {
        return $command->queue($queue, $command);
    }

    # 最后一步吧job放入队列
    return $this->pushCommandToQueue($queue, $command);
}


public function connection($name = null)
{
    # 上面传入的连接是 null，这里会去拿默认连接
    $name = $name ?: $this->getDefaultDriver(); # redis

    # 如果连接池中不存在，则实例化，显然之前没有过，所以不存在
    if (! isset($this->connections[$name])) {
        $this->connections[$name] = $this->resolve($name);

        $this->connections[$name]->setContainer($this->app);
    }

    return $this->connections[$name];
}

public function getDefaultDriver()
{
    # 'default' => env('QUEUE_CONNECTION', 'sync'), (.env 我写的是 redis)

    return $this->app['config']['queue.default'];
}

# 最终这一步返回了 RedisQueue 实例
protected function resolve($name)
{
    $config = $this->getConfig($name);

    return $this->getConnector($config['driver']) # redis
        ->connect($config)
        ->setConnectionName($name);
}

# 这一步去拿 queue.php 中的 redis 配置
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
    'queue' => 'default',
    'retry_after' => 90,
    'block_for' => null,
],
protected function getConfig($name)
{
    if (! is_null($name) && $name !== 'null') {
        return $this->app['config']["queue.connections.{$name}"];
    }

    return ['driver' => 'null'];
}

# 调用之前队列注册时的闭包
# return new RedisConnector($this->app['redis']);
# 返回一个 redis 连接实例
protected function getConnector($driver)
{
    if (! isset($this->connectors[$driver])) {
        throw new InvalidArgumentException("No connector for [$driver]");
    }

    return call_user_func($this->connectors[$driver]);
}

public function connect(array $config)
{
    return new RedisQueue(
        $this->redis, $config['queue'],
        $config['connection'] ?? $this->connection,
        $config['retry_after'] ?? 60,
        $config['block_for'] ?? null
    );
}

protected function pushCommandToQueue($queue, $command)
{
    # 如果存在 queue 和 delay 属性，那么执行 laterOn
    if (isset($command->queue, $command->delay)) {
        return $queue->laterOn($command->queue, $command->delay, $command);
    }

    # 如果存在 queue 属性 那么执行 pushOn
    if (isset($command->queue)) {
        return $queue->pushOn($command->queue, $command);
    }

    # 如果存在 delay 执行 later
    if (isset($command->delay)) {
        return $queue->later($command->delay, $command);
    }

    # 都不存在 执行 push
    return $queue->push($command);
}
```

## RedisQueue.php

```php
public function push($job, $data = '', $queue = null)
{
    # 把任务 push 到 redis 中去，返回对应的 payload，意味着 dispatch 方法有返回值
    # 返回了队列的 id /var/www/html/app/Http/Controllers/DucController.php:15:string 'A0zLZ5fwx61tGPFdbMfexrvTSDd7STP0' (length=32)
    return $this->pushRaw($this->createPayload($job, $this->getQueue($queue), $data), $queue);
}


protected function createPayload($job, $queue, $data = '')
{
    # 打包对应 payload，以 json 形式返回
    $payload = json_encode($this->createPayloadArray($job, $queue, $data));

    if (JSON_ERROR_NONE !== json_last_error()) {
        throw new InvalidPayloadException(
            'Unable to JSON encode payload. Error code: '.json_last_error()
        );
    }

    return $payload;
}

protected function createObjectPayload($job, $queue)
{
    $payload = $this->withCreatePayloadHooks($queue, [
        'displayName' => $this->getDisplayName($job),
        'job' => 'Illuminate\Queue\CallQueuedHandler@call',
        'maxTries' => $job->tries ?? null,
        'timeout' => $job->timeout ?? null,
        'timeoutAt' => $this->getJobExpiration($job),
        'data' => [
            'commandName' => $job,
            'command' => $job,
        ],
    ]);

    return array_merge($payload, [
        'data' => [
            'commandName' => get_class($job),
            'command' => serialize(clone $job),
        ],
    ]);
}

# 为队列命名，文档里没写
protected function getDisplayName($job)
{
    return method_exists($job, 'displayName')
        ? $job->displayName() : get_class($job);
}

# 获取任务的超时时间
# retryUntil 你可以返回int次数也可以返回 Carbon 实例
public function getJobExpiration($job)
{
    if (! method_exists($job, 'retryUntil') && ! isset($job->timeoutAt)) {
        return;
    }

    $expiration = $job->timeoutAt ?? $job->retryUntil();

    return $expiration instanceof DateTimeInterface
        ? $expiration->getTimestamp() : $expiration;
}
```

把队列推送到相应的redis服务器中后，就结束了，接下来应该是 `artisan queue:work` 命令发挥作用了

当我执行 `art queue:work` 的时候

```php
#!/usr/bin/env php
<?php

define('LARAVEL_START', microtime(true));

/*
|--------------------------------------------------------------------------
| Register The Auto Loader
|--------------------------------------------------------------------------
|
| Composer provides a convenient, automatically generated class loader
| for our application. We just need to utilize it! We'll require it
| into the script here so that we do not have to worry about the
| loading of any our classes "manually". Feels great to relax.
|
*/

require __DIR__.'/vendor/autoload.php';

# 可以看到这里 包含了 app.php
$app = require_once __DIR__.'/bootstrap/app.php';


# app.php
# 注册了基本的服务
# 绑定了控制台的内核
<?php

/*
|--------------------------------------------------------------------------
| Create The Application
|--------------------------------------------------------------------------
|
| The first thing we will do is create a new Laravel application instance
| which serves as the "glue" for all the components of Laravel, and is
| the IoC container for the system binding all of the various parts.
|
*/

$app = new Illuminate\Foundation\Application(
    dirname(__DIR__)
);

/*
|--------------------------------------------------------------------------
| Bind Important Interfaces
|--------------------------------------------------------------------------
|
| Next, we need to bind some important interfaces into the container so
| we will be able to resolve them when needed. The kernels serve the
| incoming requests to this application from both the web and CLI.
|
*/

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

/*
|--------------------------------------------------------------------------
| Return The Application
|--------------------------------------------------------------------------
|
| This script returns the application instance. The instance is given to
| the calling script so we can separate the building of the instances
| from the actual running of the application and sending responses.
|
*/

return $app;



# 最后执行
public function handle($input, $output = null)
{
    try {
        $this->bootstrap();

        return $this->getArtisan()->run($input, $output);
    } catch (Exception $e) {
        $this->reportException($e);

        $this->renderException($output, $e);

        return 1;
    } catch (Throwable $e) {
        $e = new FatalThrowableError($e);

        $this->reportException($e);

        $this->renderException($output, $e);

        return 1;
    }
}


# Illuminate\Foundation\Providers\ConsoleSupportServiceProvider::class, 这个服务完成了命令的注册

```

