---
title: laravel 源码解析之用户认证
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - laravel
tags:
 - 源码解析
 - 用户认证
publish: true
---



> 系统默认自带了用户认证组件，只需要执行 `php artisan make:auth`，系统就会为你生成视图文件以及注册好路由

视图文件位于 `views/auth/` 目录，系统默认添加的路由为 `Auth::routes();`

视图文件显而易见，但是到底注册了哪些路由缺不好找，

Auth Facade 中的 routes 方法

```php
public static function routes(array $options = [])
{
    static::$app->make('router')->auth($options);
}
```

`router` 实例是指向哪个实体类呢？

```php
$this->app->singleton('router', function ($app) {
    return new Router($app['events'], $app);
});

```

可以看到源码中把 `router` 和 `\Illuminate\Routing\Router::class,` 绑定了,

```php
# 可以看到auth方法能传一个数组参数，用来控制是否注册 register、reset、verify 路由
# 你可以这样传 Auth::routes['reset' => true, 'verify' => true]
public function auth(array $options = [])
{
    $this->get('login', 'Auth\LoginController@showLoginForm')->name('login');
    $this->post('login', 'Auth\LoginController@login');
    $this->post('logout', 'Auth\LoginController@logout')->name('logout');

    if ($options['register'] ?? true) {
        $this->get('register', 'Auth\RegisterController@showRegistrationForm')->name('register');
        $this->post('register', 'Auth\RegisterController@register');
    }

    if ($options['reset'] ?? true) {
        $this->resetPassword();
    }

    if ($options['verify'] ?? false) {
        $this->emailVerification();
    }
}


public function resetPassword()
{
    $this->get('password/reset', 'Auth\ForgotPasswordController@showLinkRequestForm')->name('password.request');
    $this->post('password/email', 'Auth\ForgotPasswordController@sendResetLinkEmail')->name('password.email');
    $this->get('password/reset/{token}', 'Auth\ResetPasswordController@showResetForm')->name('password.reset');
    $this->post('password/reset', 'Auth\ResetPasswordController@reset')->name('password.update');
}

public function emailVerification()
{
    $this->get('email/verify', 'Auth\VerificationController@show')->name('verification.notice');
    $this->get('email/verify/{id}', 'Auth\VerificationController@verify')->name('verification.verify');
    $this->get('email/resend', 'Auth\VerificationController@resend')->name('verification.resend');
}
```

我们先来实行登录

```php
public function login(Request $request)
{
    # 表单验证
    $this->validateLogin($request);

	# 登录速率限制
    if ($this->hasTooManyLoginAttempts($request)) {
        $this->fireLockoutEvent($request);

        return $this->sendLockoutResponse($request);
    }

    if ($this->attemptLogin($request)) {
        return $this->sendLoginResponse($request);
    }

    $this->incrementLoginAttempts($request);

    return $this->sendFailedLoginResponse($request);
}


protected function hasTooManyLoginAttempts(Request $request)
{
    return $this->limiter()->tooManyAttempts(
        $this->throttleKey($request), $this->maxAttempts()
    );
}

# 登录用户账号加ip 作为key 比如邮箱登录的话 key 就是 1@q.c|127.0.0.1 
# 此外还会存一个 1@q.c|127.0.0.1 :timer 计算剩下时间
protected function throttleKey(Request $request)
{
    return Str::lower($request->input($this->username())).'|'.$request->ip();
}

# 判断用户是否超过最大尝试次数
# $this->cache 系统会把 key(1@q.c|127.0.0.1) 存到缓存中 
public function tooManyAttempts($key, $maxAttempts)
{
    if ($this->attempts($key) >= $maxAttempts) {
        if ($this->cache->has($key.':timer')) {
            return true;
        }

        $this->resetAttempts($key);
    }

    return false;
}

# 如果你在 LoginController 中写了 maxAttempts 属性，那么以你写的为准，最大尝试次数
public function maxAttempts()
{
    return property_exists($this, 'maxAttempts') ? $this->maxAttempts : 5;
}

# decayMinutes 属性来控制，当你登录次数过多时，需要等待的时间
public function decayMinutes()
{
    return property_exists($this, 'decayMinutes') ? $this->decayMinutes : 1;
}

#########  attemptLogin  #########
protected function attemptLogin(Request $request)
{
    return $this->guard()->attempt(
        $this->credentials($request), $request->filled('remember')
    );
}

public function attempt(array $credentials = [], $remember = false)
{
    $this->fireAttemptEvent($credentials, $remember);
	# 去数据库中搜索账号相同的第一个用户，拿出来比对
    $this->lastAttempted = $user = $this->provider->retrieveByCredentials($credentials);

    # 通过 password_verify 函数判断密码是否一致
    if ($this->hasValidCredentials($user, $credentials)) {
        $this->login($user, $remember);

        return true;
    }

    $this->fireFailedEvent($user, $credentials);

    return false;
}

public function login(AuthenticatableContract $user, $remember = false)
{
    # 更新下一次返回的 session Id
    $this->updateSession($user->getAuthIdentifier());

    if ($remember) {
        $this->ensureRememberTokenIsSet($user);

        $this->queueRecallerCookie($user);
    }


    $this->fireLoginEvent($user, $remember);

    $this->setUser($user);
}

protected function updateSession($id)
{
    $this->session->put($this->getName(), $id);

    # 重置 session Id
    $this->session->migrate(true);
}

# auth 实例
$this->app->singleton('auth', function ($app) {
 
    $app['auth.loaded'] = true;

    return new AuthManager($app);
});


public function guard($name = null)
{
    # 获取默认的看守器，下文展开讲 guard
    $name = $name ?: $this->getDefaultDriver();

    return $this->guards[$name] ?? $this->guards[$name] = $this->resolve($name);
}

# 拿到配置的值 默认是 web
public function getDefaultDriver()
{
    return $this->app['config']['auth.defaults.guard'];
}

protected function resolve($name)
{
    $config = $this->getConfig($name);

    if (is_null($config)) {
        throw new InvalidArgumentException("Auth guard [{$name}] is not defined.");
    }

    if (isset($this->customCreators[$config['driver']])) {
        return $this->callCustomCreator($name, $config);
    }

    # createSessionDriver
    $driverMethod = 'create'.ucfirst($config['driver']).'Driver';

    if (method_exists($this, $driverMethod)) {
        return $this->{$driverMethod}($name, $config);
    }

    throw new InvalidArgumentException("Auth driver [{$config['driver']}] for guard [{$name}] is not defined.");
}

# 拿到         
# 'web' => [
#    'driver' => 'session',
#    'provider' => 'users',
# ],
protected function getConfig($name)
{
    return $this->app['config']["auth.guards.{$name}"];
}

public function createSessionDriver($name, $config)
{
    $provider = $this->createUserProvider($config['provider'] ?? null);

    $guard = new SessionGuard($name, $provider, $this->app['session.store']);

	# session driver 使用 session 和 cookie
    if (method_exists($guard, 'setCookieJar')) {
        $guard->setCookieJar($this->app['cookie']);
    }

    if (method_exists($guard, 'setDispatcher')) {
        $guard->setDispatcher($this->app['events']);
    }

    # 绑定 request 实例
    if (method_exists($guard, 'setRequest')) {
        $guard->setRequest($this->app->refresh('request', $guard, 'setRequest'));
    }

    return $guard;
}

public function createUserProvider($provider = null)
{
    # 拿到 provider 的配置
    #'users' => [
    #    'driver' => 'eloquent',
    #    'model' => App\User::class,
    #],
    if (is_null($config = $this->getProviderConfiguration($provider))) {
        return;
    }

    if (isset($this->customProviderCreators[$driver = ($config['driver'] ?? null)])) {
        return call_user_func(
            $this->customProviderCreators[$driver], $this->app, $config
        );
    }

    # driver 支持的类型
    switch ($driver) {
        case 'database':
            return $this->createDatabaseProvider($config);
        case 'eloquent':
            return $this->createEloquentProvider($config);
        default:
            throw new InvalidArgumentException(
                "Authentication user provider [{$driver}] is not defined."
            );
    }
}

# 创建了一个 EloquentUserProvider 实例，并且设置了
protected function createEloquentProvider($config)
{
    return new EloquentUserProvider($this->app['hash'], $config['model']);
}
```

## guard

> guard 指的是你用户账号密码登录时，所认证的表，经常用在多表登录的情况，
>
> 也就是前后台用户存不同的用户表，这时候可以指定用户的guard来控制验证的表

注意使用 `auth()->user()` 时一定要传 `guard`不然会使用你的默认 guard，即每次你获取当前用户必须这样写 `auth($guard)->user()`，还有一种更好的办法就是替换默认的guard，使用 `auth()->shouldUse($guard)` 方法（你可以在测试的 `actingAs` 方法中发现这个方法）

