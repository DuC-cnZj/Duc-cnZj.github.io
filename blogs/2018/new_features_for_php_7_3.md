---
title:  php 7.3 新特性
date: "2018-12-07 13:07:28"
sidebar: false
categories:
 - 技术
tags:
 - php7.3
publish: true
---


> php 7.3 于几天前正式推出，让我们来看一看都有什么变动。

## `is_countable` [RFC](https://wiki.php.net/rfc/is-countable)

在计算不可数对象时，PHP 7.2添加了警告。该`is_countable`功能可以帮助防止此警告。

```php
$count = is_countable($variable) ? count($variable) : null;
```

## `array_key_first`和`array_key_last` [rfc](https://wiki.php.net/rfc/array_key_first_last)

这两个函数基本上就是这个名字所说的。

```php
$array = [
    'a' => '…',
    'b' => '…',
    'c' => '…',
];

array_key_first($array); // 'a'
array_key_last($array); // 'c'
```

原来RFC还提出`array_value_first`和`array_value_last`，但这些被多数人投了反对票。

另一个想法`array_first`和`array_last`提议将返回一个元组`[$key => $value]`，但意见是混合的。现在我们只有两个函数来获取数组的第一个和最后一个键。

## 灵活的Heredoc语法[rfc](https://wiki.php.net/rfc/flexible_heredoc_nowdoc_syntaxes)

Heredoc可以成为更大字符串的有用工具，尽管它们在过去有一个缩进的怪癖。

```php
// Instead of this:

$query = <<<SQL
SELECT * 
FROM `table`
WHERE `column` = true;
SQL;

// You can do this:

$query = <<<SQL
    SELECT * 
    FROM `table`
    WHERE `column` = true;
    SQL;
```

当您在已嵌套的上下文中使用Heredoc时，这尤其有用。

所有行都将忽略关闭标记前面的空格。

一个重要的注意事项：由于这种变化，一些现有的Heredocs可能会在他们的身体中使用相同的结束标记时破坏。

```php
$str = <<<FOO
abcdefg
    FOO
FOO;

// Parse error: Invalid body indentation level in PHP 7.3
```

## 函数中的尾随逗号调用[rfc](https://wiki.php.net/rfc/trailing-comma-function-calls)

数组已经可以实现的功能，现在也可以通过函数调用完成。请注意，在功能定义中不可能！

```php
$compacted = compact(
    'posts',
    'units',
);
```

## 更好的类型错误报告

`TypeErrors`对于用于打印其全名的整数和布尔值，它已被更改为`int`和`bool`匹配代码中的类型提示。

```
Argument 1 passed to foo() must be of the type int/bool
```

与PHP 7.2相比：

```
Argument 1 passed to foo() must be of the type 
integer/boolean
```

## 可以抛出JSON错误[rfc](https://wiki.php.net/rfc/json_throw_on_error)

以前，JSON解析错误是调试的麻烦。JSON函数现在接受一个额外的选项，使它们在解析错误时抛出异常。这一变化显然增加了一个新例外：`JsonException`。

```php
json_encode($data, JSON_THROW_ON_ERROR);

json_decode("invalid json", null, 512, JSON_THROW_ON_ERROR);

// Throws JsonException
```

虽然此功能仅适用于新添加的选项，但它有可能成为未来版本的默认行为。

## `list`参考分配[rfc](https://wiki.php.net/rfc/list_reference_assignment)

将`list()`其简写`[]`语法现在支持引用。

```php
$array = [1, 2];

list($a, &$b) = $array;

$b = 3;

// $array = [1, 3];
```

## [rfc中](https://wiki.php.net/rfc/compact)未定义的变量`compact` 

传递给的未定义变量`compact`将通过通知进行报告，之前已被忽略。

```php
$a = 'foo';

compact('a', 'b'); 

// Notice: compact(): Undefined variable: b
```

## 不区分大小写的常量[rfc](https://wiki.php.net/rfc/case_insensitive_constant_deprecation)

有一些边缘情况允许使用不区分大小写的常量。这些已被弃用。

## 相同的站点cookie [rfc](https://wiki.php.net/rfc/same-site-cookie)

这种改变不仅增加了一个新的参数，它还改变了方式`setcookie`，`setrawcookie`并且`session_set_cookie_params`功能以不间断的方式工作。

它们现在支持一系列选项，同时仍然向后兼容，而不是为已经很大的功能添加了一个参数。一个例子：

```php
bool setcookie(
    string $name 
    [, string $value = "" 
    [, int $expire = 0 
    [, string $path = "" 
    [, string $domain = "" 
    [, bool $secure = false 
    [, bool $httponly = false ]]]]]] 
)

bool setcookie ( 
    string $name 
    [, string $value = "" 
    [, int $expire = 0 
    [, array $options ]]] 
)

// Both ways work.
```

## PCRE2迁移[rfc](https://wiki.php.net/rfc/pcre2-migration)

PCRE - “Perl兼容正则表达式”的缩写 - 已更新至v2。

迁移的重点是最大向后兼容性，尽管有一些重大变化。请务必阅读[RFC](https://wiki.php.net/rfc/pcre2-migration)以了解它们。

## MBString更新[README](https://github.com/php/php-src/blob/php-7.3.0RC6/UPGRADING#L186-L232)

`MBString`PHP是[处理复杂字符串](http://php.net/manual/en/intro.mbstring.php)的方式。此模块已在此版本的PHP中收到一些更新。你可以[在这里](https://github.com/php/php-src/blob/php-7.3.0RC6/UPGRADING#L186-L232)阅读它。

## 几个弃用[rfc](https://wiki.php.net/rfc/deprecations_php_7_3)

几个小东西已被弃用，因此可能会在代码中显示错误。

- 未记录的`mbstring`函数别名
- 带整数针的字符串搜索功能
- `fgetss()`功能和`string.strip_tags`过滤器
- 定义一个独立的`assert()`功能
- `FILTER_FLAG_SCHEME_REQUIRED`和`FILTER_FLAG_HOST_REQUIRED`旗帜
- `pdo_odbc.db2_instance_name` php.ini指令

有关每个弃用的完整说明，请参阅[RFC](https://wiki.php.net/rfc/deprecations_php_7_3)。