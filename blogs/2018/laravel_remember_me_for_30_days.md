---
title: laravel 记住我 30 天 ，Cookie
date: "2018-10-29 14:38:29"
sidebar: false
categories:
 - 技术
tags:
 - laravel
 - Cookie
publish: true
---


## 解决方案

```php
<?php

namespace App\Listeners;

use Cookie;
use Illuminate\Auth\Events\Login;
use Illuminate\Contracts\Cookie\QueueingFactory;

class SetCookieListener
{
    protected $cookieQueue;
    protected $cookie;

    /**
     * Create the event listener.
     *
     * @param QueueingFactory $cookieQueue
     * @param Cookie $cookie
     * @internal param QueueingFactory $cookie
     */
    public function __construct(QueueingFactory $cookieQueue, Cookie $cookie)
    {
        $this->cookieQueue = $cookieQueue;
        $this->cookie = $cookie;
    }

    /**
     * Handle the event.
     *
     * @param  Login $event
     */
    public function handle(Login $event)
    {
        $cookieName = \Auth::guard()->getRecallerName();
        $min = 30 * 24 * 60;

        if ($event->remember) {
            $value = $this->cookie->get($cookieName);
            $first_time = $this->cookie->get('first_time');
            if ($value && ! $first_time) {
                $this->cookieQueue->queue('first_time', 'changed', $min);
                $this->cookieQueue->queue($cookieName, $value, $min);
            }
        }
    }
}

```





```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Event;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        'Illuminate\Auth\Events\Login' => [
            'App\Listeners\SetCookieListener',
        ],
    ];
```



## 分析



```php
// 可以看到 中间件 RedirectIfAuthenticated 中检查登陆，调到 sessionGuard 中的 check
class RedirectIfAuthenticated
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @param  string|null  $guard
     * @return mixed
     */
    public function handle($request, Closure $next, $guard = null)
    {
        if (Auth::guard($guard)->check()) {
            return redirect('/home');
        }

        return $next($request);
    }
}

// sessionGuard 中的 check

    public function check()
    {
        return ! is_null($this->user());
    }
```



```Php
// sessionGuard 中的 user
public function user()
    {
        if ($this->loggedOut) {
            return;
        }

        // If we've already retrieved the user for the current request we can just
        // return it back immediately. We do not want to fetch the user data on
        // every call to this method because that would be tremendously slow.
        if (! is_null($this->user)) {
            return $this->user;
        }

        $id = $this->session->get($this->getName());

        // First we will try to load the user using the identifier in the session if
        // one exists. Otherwise we will check for a "remember me" cookie in this
        // request, and if one exists, attempt to retrieve the user using that.
        if (! is_null($id)) {
            if ($this->user = $this->provider->retrieveById($id)) {
                $this->fireAuthenticatedEvent($this->user);
            }
        }

        // If the user is null, but we decrypt a "recaller" cookie we can attempt to
        // pull the user data on that cookie which serves as a remember cookie on
        // the application. Once we have a user we can return it to the caller.
        $recaller = $this->recaller();

  ### 当用户登陆过期并且记住我的 cookie 值不为空的时候，去拿去 cookie 中的 user 信息，并返回 #####
        if (is_null($this->user) && ! is_null($recaller)) {
            $this->user = $this->userFromRecaller($recaller);

            if ($this->user) {
                $this->updateSession($this->user->getAuthIdentifier());
			## 触发用户登录时间 ## 👆通过监听登录时间来改变 cookie
                $this->fireLoginEvent($this->user, true);
            }
        }

        return $this->user;
    }
```



如果你「记住」用户，可以使用 `viaRemember` 方法来检查这个用户***是否使用***「记住我」 cookie 进行认证



[cookie](https://www.cnblogs.com/phpper/p/6801678.html)

[另一种记住我解决方案](http://blog.csdn.net/wangyanqi323/article/details/72476689)