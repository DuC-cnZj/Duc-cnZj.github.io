---
title: laravel 个性化语言设置
date: '2019-10-05 10:30:20'
sidebar: false
categories:
 - 技术
tags:
 - laravel
 - lang
publish: true
---


> https://laravel-news.com/testing-api-validation-errors-with-different-locales
>
> 参考上面地址



## 按照上文的配置可以实现前端个性化语言

前端选lang存到 `localstorage` ，然后在每个请求的 `header` 头中加上 `Accept-Language: zh_CN`，这时候后端就会返回中文了



## 看看 `app()->setLocale($lang);` 做了哪些事情

```php
public function setLocale($locale)
{
  $this['config']->set('app.locale', $locale);

  $this['translator']->setLocale($locale);

  $this['events']->dispatch(new Events\LocaleUpdated($locale));
}
```


```php
$events->listen(class_exists('Illuminate\Foundation\Events\LocaleUpdated') ? 'Illuminate\Foundation\Events\LocaleUpdated' : 'locale.changed', function () use ($service) {
  $service->updateLocale();
});

#### 事件触发 locate 更新
public function updateLocale()
{
  $app = $this->app && method_exists($this->app, 'getLocale') ? $this->app : app('translator');
  $locale = $app->getLocale();
  Carbon::setLocale($locale); // 用处是在你使用 diffForHumans() 方法时翻译成对应的语言
  CarbonImmutable::setLocale($locale);
  CarbonPeriod::setLocale($locale);
  CarbonInterval::setLocale($locale);

  // @codeCoverageIgnoreStart
  if (class_exists(IlluminateCarbon::class)) {
    IlluminateCarbon::setLocale($locale);
  }

  if (class_exists(Date::class)) {
    try {
      $root = Date::getFacadeRoot();
      $root->setLocale($locale);
    } catch (Throwable $e) {
      // Non Carbon class in use in Date facade
    }
  }
  // @codeCoverageIgnoreEnd
}
```

> `timezone` 和 `locate` 没啥关系



1. 设置 `app.locale` 方便别的地方能拿到全局的语言配置
2. 设置了 translator，也就是把你的 __('xxx') 翻译成了相关语言
3. 其中第三个是在你使用 diffForHumans() 方法时翻译成对应的语言



