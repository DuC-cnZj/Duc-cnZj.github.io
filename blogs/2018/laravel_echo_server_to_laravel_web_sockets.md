---
title: 从laravel-echo-server转换到laravel-websockets
date: '2021-05-12 12:54:14'
sidebar: true
categories:
 - laravel
tags:
 - laravel
 - websocket
publish: true
---



> 本文转载自 [ohdear](https://ohdear.app/blog/transitioning-from-laravel-echo-server-to-laravel-websockets)

我们刚刚从基于[laravel-echo-server](https://github.com/tlaverdure/laravel-echo-server)的websocket服务器过渡到完全由PHP驱动[的服务器](https://github.com/tlaverdure/laravel-echo-server)：[laravel-websockets](https://github.com/beyondcode/laravel-websockets)。在这篇文章中，我们将重点介绍我们为何以及如何采取这一举措。

## 简化我们的堆栈

由于我们是在[Laravel](https://laravel.com/)上[构建的](https://laravel.com/)，因此在构建阶段我们已经运行了相当多的nodejs。我们的前端JavaScript和CSS已经通过[webpack](https://laravel.com/docs/5.7/mix)编译。所以在某种程度上，我们的堆栈已经包括`node`使这一切都发生。

我们流畅的用户体验（如果我们自己这样说;-)）的一部分来自于websockets的使用，这使我们能够在他们的仪表板和我们的[主页中](https://ohdear.app/)为我们的用户提供即时反馈。为了做到这一点，我们总是使用[laravel-echo-server](https://github.com/tlaverdure/laravel-echo-server)，`node`一个websocket服务器的实现。

要使websocket服务器正常工作，您可以使用2种方法：使用Redis队列或直接通过HTTP发布消息。我们使用了Redis队列，这意味着我们发生了以下事件：

1. Laravel向Redis频道发布消息
2. echo-server侦听存储在那里的新事件
3. echo-server将这些中继到其订户/客户端

这对我们来说非常有效，没有任何问题。

但我们发现自己处于测试更简单方法的独特位置：在PHP中完全运行websocket服务器而无需节点。

## 转向PHP
我们最初的反应是“ *但是，当然，PHP无法处理Node进程的负载，对吧？* ”。

好吧，为了开始我们[使用Artillery对laravel-websockets包进行基准测试](https://ma.ttias.be/benchmarking-websocket-server-performance-with-artillery)。我们发现PHP中的websocket实现可以非常轻松地处理我们的负载并保持低于50MB的内存消耗。

在我们的测试过程中，性能与节点实现相似。

由于我们没有丢失任何东西，我们专注于删除我们的生产机器的节点依赖，并在PHP中运行整个websocket堆栈。

## 添加TLS和主管

我们的设置已经使用Nginx作为TLS代理以及Supervisor来保持所有工作人员的运行，因此我们已经有了构建块来为我们的新websocket服务器添加一些配置。

我们[配置了Nginx和Supervisor](https://ma.ttias.be/deploying-laravel-websockets-with-nginx-reverse-proxy-and-supervisord)来处理TLS部分和工作运行非常快。

## 从laravel-echo-server转换到laravel-websockets 

代码方面，改变是一块蛋糕。除非我们忘记某些事情，否则它包括：

- 删除`/js/socket.io.js`我们前端的所有引用（我们的websockets 不再需要[socket.io](https://socket.io/docs/client-api/)）
- 安装[laravel-websockets](https://github.com/beyondcode/laravel-websockets)并按照其安装说明进行操作
- 将我们的前端代码从使用socket.io更改为pusher

```js
     window.Echo = new Echo({
-        broadcaster: 'socket.io',
+        broadcaster: 'pusher',
         key: window.pusherKey,
-        host: window.pusherHost,
+        wsHost: window.pusherHost,
+        wsPort: window.pusherPort,
+        disableStats: true,
     });
```

- 将我们的广播驱动程序从Redis更改为使用Pusher `.env`

```env
- BROADCAST_DRIVER=redis
+ BROADCAST_DRIVER=pusher
+ PUSHER_ENDPOINT=socket.ohdear.app
+ PUSHER_ENDPOINT_SCHEME=https
```

这足以让我们转向[laravel-websockets](https://github.com/beyondcode/laravel-websockets)。

## 我们获得了什么？

其最大的收获之一是在我们的开发过程中：我们现在只需要运行`php artisan websocket:serve`以获得本地websocket服务器，而不必处理laravel-echo-server的（相当令人困惑的）配置。

此外，我们简化了服务器设置，现在完全依赖没有Node的PHP来运行websockets。管理较少的软件总是一个胜利，特别是从安全角度（跟踪节点生态系统和echo-server的依赖关系）。

对我们来说，进行转换是明智的选择。