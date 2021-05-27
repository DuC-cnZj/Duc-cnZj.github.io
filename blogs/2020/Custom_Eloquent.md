---
title: Using Custom Eloquent Casts in Laravel 7
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - laravel
tags:
 - Eloquent
 - laravel
publish: true
---


> 原文 https://atymic.dev/blog/laravel-custom-casts/

Laravel 7 is pretty much upon us (scheduled for release on 3rd March) and brings a bunch of awesome new features & improvements. One of these ones I'm most looking forward to is [Custom Eloquent Casts](https://github.com/laravel/framework/pull/31035).

Historically you've been limited to the default set of casts provided by Laravel, which cover basic language type plus dates. While there are some existing packages out there implementing custom casting they have a major drawback. Because they override the `setAttribute` and `getAttribute` methods through a trait, they can't be used with any other packages that also override those methods.

Now that Laravel 7 supports these casts natively there won't be compatibility issues between libraries!

## How custom eloquent casts work

Any object implementing the new `CastsAttributes` contract provided by Laravel can now be used in the `$casts` property on your model. When accessing properties on your model, Eloquent will first check if there's a custom cast to transform the value with before returning the value to you.

Keep in mind that your cast will be called on **every single get and set operation on the model**, so consider caching intensive operations.

## Creating Custom Casts

A popular feature suggestion for Laravel has been allowing selective encryption of model properties, and with custom casts this is super simple to implement.

```php
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;

class EncryptCast implements CastsAttributes
{
    public function get($model, $key, $value, $attributes)
    {
        return encrypt($value);
    }

    public function set($model, $key, $value, $attributes)
    {
        return decrypt($value);
    }
}
```

In your model, you can then assign a property to the cast we just created.

```php
class TestModel extends Model
{
    /**
     * The attributes that should be cast to native types.
     *
     * @var array
     */
    protected $casts = [
        'secret' => EncryptCast::class,
    ];
}
```

Now that we've set it up, let's test it out! Because the `encrypt`/`decrypt` functions serialize input, you can store pretty much anything (but you should probably stick to the simple built-in types).

```php
$model = new TestModel();

$model->secret = 'Hello World'; // Plain text value

// Encrypted value (which will be saved to the database)
// Raw Value: eyJpdiI6InV4Q25ZN0FZUW5YSEZkRCtZSGlVXC9BPT0iLCJ2Y...
$model->getAttributes()['secret'];

$model->save(); // Save & reload the model from the DB
$model->fresh();

echo $model->secret; // Hello World
```

Because custom casts are just objects, they can be as simple or complex as required. There's a couple of additional features in the implementation, including "Inbound" casts (that only cast set values) and the ability to accept config from the deceleration in the model's `$casts`.

For example, we could limit strings to a configurable length, but only when setting values (therefore any existing values would remain the same length).

```php
class LimitCaster implements CastsInboundAttributes
{
    public function __construct($length = 25)
    {
        $this->length = $length;
    }

    public function set($model, $key, $value, $attributes)
    {
        return [$key => Str::limit((string) $value, $this->length)];
    }
}
```

In the model, separate the class name of the cast and the parameters with a colon.

```php
class TestModel extends Model
{
    protected $casts = [
        'limited_str' => LimitCaster::class . ':10', // Limit to 10 characters
    ];
}
```

## Wrapping Up

I'm really excited to see some of the awesome casts the community comes up with once Laravel 7 is out. It's super easy to publish custom casts as in packages as well, so hopefully we'll see a bunch of packages proving custom casts in the future.

One of the ones I've missed on previous projects is a `DateInterval` or `CarbonInterval` cast, so I've [published a package](https://github.com/atymic/laravel-dateinterval-cast) in case anyone else has come across the same problem.

Got any questions? Feel free to comment below and I'll do my best to answer them!