---
title: ip
date: "2019-01-10 16:15:05"
sidebar: true
categories:
 - 技术
tags:
 - package
 - php
publish: true
---

> A SDK for ip. https://github.com/DuC-cnZj/ip

## Installing

```shell
$ composer require duc_cnzj/ip -vvv
```

## Usage

```php
<?php
...

$client = new IpClient();

$client->setIp('xxx.xxx.xxx.xxx')->getCity();
$client->setIp('xxx.xxx.xxx.xxx')->getCountry();
$client->setIp('xxx.xxx.xxx.xxx')->getRegion();
$client->setIp('xxx.xxx.xxx.xxx')->getIp();

# or
$client->setIp('xxx.xxx.xxx.xxx');
$client->getCity();
$client->city;

# use Provider (baidu, taobao)
$client = new IpClient();

# if use baidu api, you need provide ak secret.
$client->useProvider('baidu')
    ->setProviderConfig('baidu', ['ak' => 'xxxxxxxxxxxx']);

# if use taobao.
$client->useProvider('taobao');

# default use both baidu and taobao. if you want use both, pls set baidu ak secret.
$client->setProviderConfig('baidu', ['ak' => 'xxxxxxxxxxxx']);

# package will try 3 times when provider response error. use try to reset tryTimes.
$client->setIp('xxx.xxx.xxx.xxx')->try(10);

# in laravel
 $client = app('ip');
```

### extra property
```php
<?php
...

# baidu return point_x and point_y, so you can:
$client->useProvider('baidu')
    ->setProviderConfig('baidu', ['ak' => 'xxxxxxxxxxxx']);

$client->getPointX();
$client->getPointY();

# taobao return isp, so u can:

$client->useProvider('taobao')
    ->setIp('xxx.xxx.xxx.xxx')
    ->getIsp();
```

when getXXX() false it will return null, u can use `$client->getErrors()` get error info;

### alias
```php
$client->use('taobao'); 
# Equals to
$client->useProvider('taobao'); 
```


other methods
```php
$client->cliearUse(); # will clear providers;

# custom your instance. need implement DucCnzj\Ip\Imp\IpImp;
$client->bound('taobao', $taobao);

# set custom CacheStore. default is ArrayStore. need implement DucCnzj\Ip\Imp\CacheStoreImp;
$client->setCacheStore($store);
```


## License

MIT
