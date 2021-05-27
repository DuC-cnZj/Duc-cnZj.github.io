---
title: 如何安装php 7.3
date: '2021-05-12 12:54:14'
sidebar: true
categories:
 - php
tags:
 - php
publish: true
---


![How to install PHP 7.3](../images/2018_12_11_DWN13rtW3B.png)
[PHP 7.3已经发布](https://secure.php.net/releases/7_3_0.php)，带来一些伟大的新功能，语言，比如[尾随在函数调用逗号](https://wiki.php.net/rfc/trailing-comma-function-calls)，[当JSON解析失败引发错误](https://wiki.php.net/rfc/json_throw_on_error)，[`array_key_first()`/ `array_key_last()`功能](https://wiki.php.net/rfc/array_key_first_last)，以及[更多](https://secure.php.net/manual/en/migration73.php)！

这是关于如何在Linux，Windows和OS X上安装PHP 7.3的简要指南：

## Ubuntu的

可以通过添加[OndřejSurý的PPA](https://launchpad.net/~ondrej/+archive/ubuntu/php)来安装PHP 7.3和常用扩展：

```bash
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update
```

然后，您可以使用所有常见扩展和SAPI安装PHP 7.3，如下所示：

```bash
sudo apt-get install php7.3
```

或者您可以指定这样的单个包：

```bash
sudo apt-get install php7.3-cli php7.3-fpm php7.3-bcmath php7.3-curl php7.3-gd php7.3-intl php7.3-json php7.3-mbstring php7.3-mysql php7.3-opcache php7.3-sqlite3 php7.3-xml php7.3-zip
```

支持：

- Ubuntu 14.04 (Trusty)
- Ubuntu 16.04 (Xenial)
- Ubuntu 18.04 (Bionic)
- Ubuntu 18.10 (Cosmic)

## Debian的

[OndřejSurý还](https://packages.sury.org/php/)为**Debian 8（Jessie）**和**Debian 9（Stretch）**[提供PHP 7.3软件包](https://packages.sury.org/php/)。使用以下命令添加存储库：

```bash
sudo apt-get install -y apt-transport-https lsb-release ca-certificates wget
sudo wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list
sudo apt-get update
```

然后安装PHP 7.3，其默认值如下：

```bash
sudo apt-get install php7.3
```

或者列出您想要的确切包裹：

```bash
sudo apt-get install php7.3-cli php7.3-fpm php7.3-bcmath php7.3-curl php7.3-gd php7.3-intl php7.3-json php7.3-mbstring php7.3-mysql php7.3-opcache php7.3-sqlite3 php7.3-xml php7.3-zip
```

## CentOS / RHEL和Fedora

[Remi的RPM存储库](https://rpms.remirepo.net/)具有最新PHP 7.3版本的RPM。有关如何最好地安装此项，请参阅[Remi的博客文章](https://blog.remirepo.net/post/2018/12/06/PHP-version-7.3.0-is-released)，具体取决于您的操作系统版本'跑步。

## Mac OS X.

使用[Liip `php-osx`工具](https://php-osx.liip.ch/)可以轻松安装PHP 7.3 - 只需在终端中运行以下命令：

```bash
curl -s https://php-osx.liip.ch/install.sh | bash -s 7.3
```

或者，如果您更喜欢使用Homebrew：

```bash
brew install php73
```

## Windows

Windows用户可以在[windows.php.net](https://windows.php.net/download/#php-7.3)网站上找到PHP 7.3发行版。

有关安装发行版的其他说明，请参阅此博客文章：[//kizu514.com/blog/install-php7-and-composer-on-windows-10/](http://kizu514.com/blog/install-php7-and-composer-on-windows-10/)

## phpbrew

[phpbrew](https://github.com/phpbrew/phpbrew)是一个了不起的工具，可以帮助您在计算机上下载，编译和管理多个版本的PHP。假设您已经按照[安装说明进行操作](https://github.com/phpbrew/phpbrew#requirement)并`phpbrew`启动并运行，则可以使用两个简单命令安装PHP 7.3.0：

```bash
phpbrew update
phpbrew install -j $(nproc) 7.3.0 +default
```

## docker

[官方PHP图像可以在Docker Hub上找到](https://hub.docker.com/_/php/)。使用`php:7.3`标记作为基本图像。

Docker也是在本地交互式shell中修改PHP 7.3而不先安装它的好方法！只需在您的终端中运行：

```bash
docker run -it --rm php:7.3
```