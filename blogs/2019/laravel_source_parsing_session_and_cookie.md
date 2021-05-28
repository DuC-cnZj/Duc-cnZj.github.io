---
title: laravel 源码解析之session和cookie
date: "2019-01-19 12:04:05"
sidebar: true
categories:
 - 技术
tags:
 - laravel
 - 源码解析
 - session和cookie
publish: true
---


> session和cookie概念部分转自 https://harttle.land/2015/08/10/cookie-session.html

http 协议是无状态的，也就是说客户端和服务端在打交道时，HTTP服务器并不知道这个用户是谁、是否登录过等。那么为什么现在的服务器能知道我们是否已经登录呢？这就是session和cookie的产生原因。

## Cookie 的实现机制

[Cookie](https://harttle.land/assets/img/blog/cookie.png)是由客户端保存的小型文本文件，其内容为一系列的键值对。 **Cookie是由HTTP服务器设置的，保存在浏览器中**， 在用户访问其他页面时，会在HTTP请求中附上该服务器之前设置的Cookie。 Cookie的实现标准定义在[RFC2109: HTTP State Management Mechanism](https://www.ietf.org/rfc/rfc2109.txt)中。 那么Cookie是怎样工作的呢？下面给出整个Cookie的传递流程：

1. 浏览器向某个URL发起HTTP请求（可以是任何请求，比如GET一个页面、POST一个登录表单等）

2. 对应的服务器收到该HTTP请求，并计算应当返回给浏览器的HTTP响应。

   > HTTP响应包括请求头和请求体两部分，可以参见：[读 HTTP 协议](https://harttle.land/2014/10/01/http.html)。

3. 在响应头加入`Set-Cookie`字段，它的值是要设置的Cookie。

   在[RFC2109 6.3 Implementation Limits](https://www.ietf.org/rfc/rfc2109.txt)中提到： UserAgent（浏览器就是一种用户代理）至少应支持300项Cookie， 每项至少应支持到4096字节，每个域名至少支持20项Cookie。

4. 浏览器收到来自服务器的HTTP响应。

5. 浏览器在响应头中发现`Set-Cookie`字段，就会将该字段的值保存在内存或者硬盘中。

   `Set-Cookie`字段的值可以是很多项Cookie，每一项都可以指定过期时间`Expires`。 默认的过期时间是用户关闭浏览器时。

6. 浏览器下次给该服务器发送HTTP请求时， 会将服务器设置的Cookie附加在HTTP请求的头字段`Cookie`中。

   浏览器可以存储多个域名下的Cookie，但只发送当前请求的域名曾经指定的Cookie， 这个域名也可以在`Set-Cookie`字段中指定）。

7. 服务器收到这个HTTP请求，发现请求头中有`Cookie`字段， 便知道之前就和这个用户打过交道了。

8. 过期的Cookie会被浏览器删除。

总之，服务器通过`Set-Cookie`响应头字段来指示浏览器保存Cookie， 浏览器通过`Cookie`请求头字段来告诉服务器之前的状态。 Cookie中包含若干个键值对，每个键值对可以设置过期时间。

## Cookie 的安全隐患

cookie是存在安全隐患的，因为cookie可以被篡改。

Cookie提供了一种手段使得HTTP请求可以附加当前状态， 现今的网站也是靠Cookie来标识用户的登录状态的：

1. 用户提交用户名和密码的表单，这通常是一个POST HTTP请求。
2. 服务器验证用户名与密码，如果合法则返回200（OK）并设置`Set-Cookie`为`authed=true`。
3. 浏览器存储该Cookie。
4. 浏览器发送请求时，设置`Cookie`字段为`authed=true`。
5. 服务器收到第二次请求，从`Cookie`字段得知该用户已经登录。 按照已登录用户的权限来处理此次请求。

由上面的操作可以看出，cookie不安全。这就引入了session。

## Session 的实现机制

Session 是存储在服务器端的，避免了在客户端Cookie中存储敏感数据。 Session 可以存储在HTTP服务器的内存中，也可以存在内存数据库（如redis）中， 对于重量级的应用甚至可以存储在数据库中。

我们以存储在redis中的Session为例，还是考察如何验证用户登录状态的问题。

1. 用户提交包含用户名和密码的表单，发送HTTP请求。

2. 服务器验证用户发来的用户名密码。

3. 如果正确则把当前用户名（通常是用户对象）存储到redis中，并生成它在redis中的ID。

   这个ID称为Session ID，通过Session ID可以从Redis中取出对应的用户对象， 敏感数据（比如`authed=true`）都存储在这个用户对象中。

4. 设置Cookie为`sessionId=xxxxxx|checksum`并发送HTTP响应， 仍然为每一项Cookie都设置签名。

5. 用户收到HTTP响应后，便看不到任何敏感数据了。在此后的请求中发送该Cookie给服务器。

6. 服务器收到此后的HTTP请求后，发现Cookie中有SessionID，进行放篡改验证。

7. 如果通过了验证，根据该ID从Redis中取出对应的用户对象， 查看该对象的状态并继续执行业务逻辑。

Web应用框架都会实现上述过程，在Web应用中可以直接获得当前用户。 相当于**在HTTP协议之上，通过Cookie实现了持久的会话。这个会话便称为Session。**

## 区别

1. Cookie 在客户端（浏览器），Session 在服务器端。

2. Cookie的安全性一般，他人可通过分析存放在本地的Cookie并进行Cookie欺骗。在安全性第一的前提下，选择Session更优。重要交互信息比如权限等就要放在Session中，一般的信息记录放Cookie就好了。
3.  单个Cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个Cookie。 
4. Session 可以放在 文件、数据库或内存中，比如在使用Node时将Session保存在redis中。由于一定时间内它是保存在服务器上的，当访问增多时，会较大地占用服务器的性能。考虑到减轻服务器性能方面，应当适时使用Cookie。
5.  Session 的运行依赖Session ID，而 Session ID 是存在 Cookie 中的，也就是说，如果浏览器禁用了 Cookie，Session 也会失效（但是可以通过其它方式实现，比如在 url 中传递 Session ID）。 
6. 用户验证这种场合一般会用 Session。因此，维持一个会话的核心就是客户端的唯一标识，即Session ID。



## Laravel session机制

> 正是由于cookie的不安全，所以 laravel 采用的是 session 存储。也就是返回一个session_id给客户端。
>
> laravel 没有使用 cookie 去保存用户的敏感信息。

### 首先我们先登录一个用户

![2019_01_19_yQAAf23yXw.png](../images/2019_01_19_yQAAf23yXw.png)
登录之后可以发现，cookie中多了两条信息，然后再刷新页面可以发现我们一直处于登录状态。

接下来我们来分析系统是如何识别我们是谁的。

先来看请求头，可以发现刷新之后，我们的请求会自动的把cookie发送给服务器。

```http
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cache-Control: max-age=0
Connection: keep-alive
Cookie: XSRF-TOKEN=eyJpdiI6IjIrcUNLdDIyazdLbWNsb1h2a0xpUWc9PSIsInZhbHVlIjoiRG1WdTc4bjlISmx4b21LZllVRlFZc0JMcWwrXC9jUzR5Ujd5TGVsSktlYmZ5TE5rSkdvZUNBTU1jOUdocE94UDUiLCJtYWMiOiJkN2RjMDc2ZjEyOTQwNGI4NDk1YjNmOTkxZDRjY2UwYTNiNGZlZTVhYjdmYTRiYzA1NzFhYzJiNDRlMWQ4ZGU0In0%3D; laravel_session=eyJpdiI6IjFLK1Q3cHZzUWR0SDVFOXFXRlNPcXc9PSIsInZhbHVlIjoiWUJob3VoaGxDa3UrWlJ6OW9hVytONXdScVZrRUFFWWlzNkEwS25QXC9RdmFaMWtNMVFWd2xJREZRTnlFb1wvUUM5IiwibWFjIjoiYTMyYzVlMzNkM2NkYzFjYTIxOThhMzlmZGI5ZGU2YzJmNTYxZDNjMTBiYjEzMmQ5MWJhMTQyMTE2M2JmMGExMCJ9
Host: localhost:9999
Referer: http://localhost:9999/login
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36
```

服务器接收请求并且处理，由之前的Pipeline可知，在执行控制器里面的行为之前，我们要走中间件，而 /login 路由走的是web 中间件，所以我们先来看web中间件，除了先要进过全局中间件以外，还要进过web中间件，下面是默认的web中间件

```php
'web' => [
    \App\Http\Middleware\EncryptCookies::class,
    \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
    \Illuminate\Session\Middleware\StartSession::class,
    // \Illuminate\Session\Middleware\AuthenticateSession::class,
    \Illuminate\View\Middleware\ShareErrorsFromSession::class,
    \App\Http\Middleware\VerifyCsrfToken::class,
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

首先是 EncryptCookies 中间件，它对response的cookie做了加密，使用后置中间件的写法，在请求结束后，对服务器响应头进行操作

```php

public function handle($request, Closure $next)
{
    # 这里注意了，比较难理解
    # $this->decrypt($request) 这一步是前置中间件的操作，它把你的请求中的cookie解码了，本来你的cookie是加密的，走到这里就被解密了
    # $next($this->decrypt($request)) 
    # $this->encrypt(...) 这里是后置中间件键，拆开来会比较好理解
    return $this->encrypt($next($this->decrypt($request)));
}
```

接着是    \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,也是采用后置中间件的写法，添加在处理逻辑时需要返回给客户端的cookie

```php
public function handle($request, Closure $next)
{
    $response = $next($request);

    foreach ($this->cookies->getQueuedCookies() as $cookie) {
        $response->headers->setCookie($cookie);
    }

    return $response;
}
```

接着才是前置中间件    \Illuminate\Session\Middleware\StartSession::class,

```php
public function handle($request, Closure $next)
{
    $this->sessionHandled = true;

	# 开启session
    # 首先去拿config中的session的配置，显然能拿到
    # 然后去startSession
    if ($this->sessionConfigured()) {
				# 把 返回的session 也就是 （Store 实例）赋值给 $request 的 session 属性
        $request->setLaravelSession(
            $session = $this->startSession($request)
        );

        $this->collectGarbage($session);
    }

    $response = $next($request);

	# 后置写法
    if ($this->sessionConfigured()) {
        $this->storeCurrentUrl($request, $session);

        $this->addCookieToResponse($response, $session);
    }

    return $response;
}
```

```php
protected function startSession(Request $request)
{
    return tap($this->getSession($request), function ($session) use ($request) {
        # 这一步它把request实例给到对应的handler也就是刚刚的CacheBasedSessionHandler 里面，不过对于redis来说这一步是不需要的，只有CookieSessionHandler 实例才需要，(未深究)
        $session->setRequestOnHandler($request);

        # 加载session
        $session->start();
    });
}

public function getSession(Request $request)
{
    # $this->manager->driver() 去获取配置的sessionManager，我这里用了redis，它最后返回Store实例，这个实例设置了客户端记录session_id的cookie的键名(laravel_session)和session的储存处理器(redis)
    # 
    return tap($this->manager->driver(), function ($session) use ($request) {
        # 这里 它去获取客户端中储存的session_id的cookie值
        # 这里看到它用了$session->getName()，刚刚上面所说的laravel_session就在这里获取到了
        $session->setId($request->cookies->get($session->getName()));
    });
}
# 所以这一步的操作是把客户端请求头的session_id添加到了Store中的id里面去，并且用tap方法返回了该Store


# 这里是具体的session CacheHandler 的创建，缓存前缀一些基础的 配置在这生成
protected function createCacheHandler($driver)
{
    $store = $this->app['config']->get('session.store') ?: $driver;

    return new CacheBasedSessionHandler(
        clone $this->app['cache']->store($store),
        $this->app['config']['session.lifetime']
    );
}
```

```php
public function start()
{
    $this->loadSession();

    if (! $this->has('_token')) {
        $this->regenerateToken();
    }

    # 表示session启动成功
    return $this->started = true;
}

protected function loadSession()
{
    $this->attributes = array_merge($this->attributes, $this->readFromHandler());
}

protected function readFromHandler()
{
    # 这里看到它调用了 CacheBasedSessionHandler 的 read 方法
    # read 调用了 RedisStore 中的get方法为对应的 id 加上了前缀
    if ($data = $this->handler->read($this->getId())) {
        # redis 中的数据是序列化过的，所以这里要反序列化
        # redis 存了
        # array:4 [▼
          "_token" => "rcEB96T99YH5ITqrLK6MQcWYYc2GsCn3eQUrVqPA"
          "login_web_59ba36addc2b2f9401580f014c7f58ea4e30989d" => 2
          "_flash" => array:2 [▼
            "old" => []
            "new" => []
          ]
          "_previous" => array:1 [▼
            "url" => "http://localhost:9999/home"
          ]
        ]
              
		# 上面这些就是session的内容，其中login_web_59ba36addc2b2f9401580f014c7f58ea4e30989d对应的2就是用户的id
        $data = @unserialize($this->prepareForUnserialize($data));

        if ($data !== false && ! is_null($data) && is_array($data)) {
            return $data;
        }
    }

    return [];
}
```

此后，你拿到的request实例中session属性就有了用户信息，再之后 auth 中间件判断是否登陆其实拿的就是 session 中的信息，而 session 在上面就实例化过了，并且是单例！

### remember_me

如果用户的的login_web_59ba36addc2b2f9401580f014c7f58ea4e30989d过期了，但是存在 remember_me 的token，那么内部会去解析该token，id|remember_token|password token的组成三部分，分别用|隔开，系统根据id查用户，然后校验remember_token


### logoutOtherDevices 功能

登出其他设备

1. 重置了密码，这一步感觉没必要，因为并没有校验密码，可能是出于安全性
2. 重新生成session_id