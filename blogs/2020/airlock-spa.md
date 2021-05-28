---
title: laravel (airlock/sanctum) spa 用户认证
date: '2020-01-13 16:54:14'
sidebar: false
categories:
 - 技术
tags:
 - airlock/sanctum
 - spa
 - laravel
publish: true
---


可以直接看 https://github.com/qirolab/laravel-sanctum-example 很不错

其中 这一段代码很好

> 因为每次请求会返回csrf，所以不用每次发起 post请求都去拿token，因为上一次的返回会返回新的token，只有本地拿不到的时候才去拿

```js

import Api from "./Api";
import Cookie from "js-cookie";

export default {
  getCookie() {
    let token = Cookie.get("XSRF-TOKEN");

    if (token) {
      return new Promise(resolve => {
        resolve(token);
      });
    }

    return Api.get("/csrf-cookie");
  }
};
```

详细流程

发起 post 请求 -> 获取本地csrf-token，没有拿到，然后访问远程获取token -> 请求返回会自己设置新的 csrf-token -> 第二次 post -> 获取本地csrf-token，拿到了直接发起请求 -> 返回结果设置新的 csrf-token -> 本地csrf-token 过期后 -> 发起 post 请求 -> 获取本地csrf-token，没有拿到，然后访问远程获取token -> 获取返回结果，设置新token



思考

如果本地的csrf-token 没过期，但是服务器出现意外，session全丢了，那么就算带上本地的csrf，服务器也会419，这时候不会返回新的token，这时候需要前端检查状态码，如果419则清除本地csrf-token

```php
public function handle($request, Closure $next)
{
  if (
    $this->isReading($request) ||
    $this->runningUnitTests() ||
    $this->inExceptArray($request) ||
    $this->tokensMatch($request)
  ) {
    return tap($next($request), function ($response) use ($request) {
      if ($this->shouldAddXsrfTokenCookie()) {
        // 新的token在这里返回，出现419的时候不进入这里
        $this->addCookieToResponse($request, $response);
      }
    });
  }

  throw new TokenMismatchException('CSRF token mismatch.');
}
```

```js
Api.interceptors.response.use(function (response) {
    return response;
  }, function (error) {
    if (error.response.status === 419) {
      Cookie.remove("XSRF-TOKEN")
    }
    return Promise.reject(error);
  });
```



对比 api 和 cookie-seesion



Laravel cookie/session 中是根据客户端的 session_id 获取用户信息的，并且 csrf-token 是直接服务端页面返回的



Api (sanctum) 因为拿不到csrf-token，所以需要 单独访问 /sanctum/csrf-cookie, 为了能够在本地cookie中存储设置csrf-token，必须开启 with-credentials = true, 这样客户端就可以保存服务端返回的 cookie。最后api认证是通过 Authorization 的header头，获取当前用户，和非前后端分离有所区别.看了下源码，sanctum也是先拿cookie做校验，拿不到才去拿token，所以如果cookie没过期，你其实可以不传token



XSS： 通过客户端脚本语言（最常见如：JavaScript）
在一个论坛发帖中发布一段恶意的JavaScript代码就是脚本注入，如果这个代码内容有请求外部服务器，那么就叫做XSS！

CSRF：又称XSRF，冒充用户发起请求（在用户不知情的情况下）,完成一些违背用户意愿的请求（如恶意发帖，删帖，改密码，发邮件等）。

CSRF的一个特征是，攻击者无法直接窃取到用户的信息（Cookie，Header，网站内容等），仅仅是冒用Cookie中的信息。

为什么 csrf token 能防止 csrf/xss 攻击 👉 https://juejin.im/post/5bc009996fb9a05d0a055192

----- 分割线 -------

> 参考 https://xueyuanjun.com/post/21395

## 使用

创建一个 cors middleware

> https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS
>
> Access-Control-Allow-Credentials 设置 true之后就不能 Access-Control-Allow-Origin 设置为 *

```php
<?php

namespace App\Http\Middleware;

use Closure;

class Cors
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        $h = $request->headers->get('Access-Control-Request-Headers');

        return $next($request)
            ->header('Access-Control-Allow-Origin', 'http://localhost:8080')
            ->header('Access-Control-Allow-Headers', $h)
            ->header('Access-Control-Allow-Credentials', 'true')
            ->header('Access-Control-Request-Method', '*');
    }
}

```

Web.php

```php
Route::post('/airlock/token', function (Request $request) {
    // 方便起见这样写，别模仿~
    $user = factory(User::class)->create();

    return $user->createToken('mobile')->plainTextToken;
});
```

js

```php
   onSubmit() {
     	// 加这个 带上 cookie 不然会一直 419
      axios.defaults.withCredentials = true;
      axios.get("http://localhost:8001/airlock/csrf-cookie").then(response => {
        // eslint-disable-next-line no-console
        console.log(response);

        axios
          .post("http://localhost:8001/airlock/token")
          .then(response => {
            // eslint-disable-next-line no-console
            console.log("认证成功");
            let authToken = response.data;
            axios.defaults.headers.common[
              "Authorization"
            ] = `Bearer ${authToken}`;
            axios.get("http://localhost:8001/api/user").then(response => {
              // 打印用户信息
              // eslint-disable-next-line no-console
              console.log(response.data);
            });
          })
          .catch(error => {
            // 登录失败，打印错误信息
            // eslint-disable-next-line no-console
            console.log(error);
          });
      });
    }
  }
```





源码阅读



```php
public function __invoke(Request $request)
{
  // 如果该用户web登录，那么直接返回用户，并且有最大权限？can() 方法固定返回 true
  if ($user = $this->auth->guard('web')->user()) {
    return $this->supportsTokens()
      ? $user->withAccessToken(new TransientToken)
      : $user;
  }

  
  // 如果没有web guard 登录，那么取 bearer token 去数据库查
  if ($this->supportsTokens() && $token = $request->bearerToken()) {
    $model = Airlock::$personalAccessTokenModel;

    $accessToken = $model::where('token', hash('sha256', $token))->first();

    // 过期时间不是直接存在数据库中，而是直接拿 created_at 和 配置的时间去比对，更加灵活
    if (! $accessToken ||
        ($this->expiration &&
         $accessToken->created_at->lte(now()->subMinutes($this->expiration)))) {
      return;
    }

    return $accessToken->user->withAccessToken(
      tap($accessToken->forceFill(['last_used_at' => now()]))->save()
    );
  }
}
```


