---
title: 链路追踪
date: '2020-11-10 11:23:13'
sidebar: false
categories:
 - 技术
tags:
 - 链路追踪
 - golang
 - php
publish: true
---

## php 集成 jaeger

> jukylin/jaeger-php

laravel middleware

```php
$this->app->singleton(Trace::class);
```




```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;
use Illuminate\Support\Str;
use Jaeger\Config;
use Jaeger\Span;
use Jaeger\SpanContext;

class Trace
{
    /**
     * @var Span
     */
    protected $serverSpan;
    protected $config;

    public function handle(Request $request, Closure $next)
    {
        $config = Config::getInstance();
        $config::$propagator = \Jaeger\Constants\PROPAGATOR_JAEGER;
        $tracer = $config->initTracer('php client', env("JAEGER_HOST"));
        $operationName = $request->route()->getAction('uses') instanceof Closure
            ? "closure"
            : $request->route()->getAction('uses');
        if (isset($_SERVER['UBER-TRACE-ID'])) {
            $_SERVER['UBER-TRACE-ID'] = [];
            $spanContext = $tracer->extract(\OpenTracing\Formats\TEXT_MAP, $_SERVER);
            $this->serverSpan = $tracer->startSpan($operationName, ['child_of' => $spanContext]);
        } else {
            $this->serverSpan = $tracer->startSpan($operationName);
        }

        $tracer->inject($this->serverSpan->spanContext, \OpenTracing\Formats\TEXT_MAP, $_SERVER);
        $this->config = $config;

        return $next($request);
    }

    public function terminate($request, $response)
    {
        if (!$response->isSuccessful()) {
            $this->serverSpan->setTag("error", true);
            $this->serverSpan->log(['errorData' => Str::limit($response->content(), 500)]);
            info("trace");
        }
        $this->serverSpan->finish();
        $this->config->flush();
    }
}

```

Golang

> https://blog.csdn.net/liyunlong41/article/details/88043604
>
> https://segmentfault.com/a/1190000014546372
>
> https://blog.csdn.net/moxiaomomo/article/details/80721328


## grpc

> https://github.com/DuC-cnZj/rpc-facades-generator

```php
// grpc
OrderClient::list([], true, ['UBER-TRACE-ID' => [$_SERVER['UBER-TRACE-ID']]])
```

