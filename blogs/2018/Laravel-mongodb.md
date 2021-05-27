---
title: Laravel-mongodb 消息通知
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - laravel
tags:
 - laravel
 - mongodb
publish: true
---


## 起因☠️

**Illuminate\Notifications\Notifiable**   中的  `HasDatabaseNotifications.php`  写死了 `DatabaseNotification.php`  的位置，导入我们无法更改其数据库的连接，把   `mongodb`  设置成默认连接好像也不管用，所以现在我们要用 `mongodb` 来实现消息通知，那么必须对其进行重写，覆盖



## 具体做法😕

手动创建三个类，继承 `Jenssegers\Mongodb\Eloquent\Model` ，添加 ` protected $connection = 'mongodb';` 

```php
<?php

namespace App\Overrides\Notifications;

use Jenssegers\Mongodb\Eloquent\Model;
use Illuminate\Notifications\DatabaseNotificationCollection;

class DatabaseNotification extends Model
{
    protected $connection = 'mongodb';

    /**
     * Indicates if the IDs are auto-incrementing.
     *
     * @var bool
     */
    public $incrementing = false;

    /**
     * The table associated with the model.
     *
     * @var string
     */
    protected $table = 'notifications';

    /**
     * The guarded attributes on the model.
     *
     * @var array
     */
    protected $guarded = [];

    /**
     * The attributes that should be cast to native types.
     *
     * @var array
     */
    protected $casts = [
        'data'    => 'array',
        'read_at' => 'datetime',
    ];

    /**
     * Get the notifiable entity that the notification belongs to.
     */
    public function notifiable()
    {
        return $this->morphTo();
    }

    /**
     * Mark the notification as read.
     *
     * @return void
     */
    public function markAsRead()
    {
        if (is_null($this->read_at)) {
            $this->forceFill(['read_at' => $this->freshTimestamp()])->save();
        }
    }

    /**
     * Mark the notification as unread.
     *
     * @return void
     */
    public function markAsUnread()
    {
        if (! is_null($this->read_at)) {
            $this->forceFill(['read_at' => null])->save();
        }
    }

    /**
     * Determine if a notification has been read.
     *
     * @return bool
     */
    public function read()
    {
        return $this->read_at !== null;
    }

    /**
     * Determine if a notification has not been read.
     *
     * @return bool
     */
    public function unread()
    {
        return $this->read_at === null;
    }

    /**
     * Create a new database notification collection instance.
     *
     * @param  array  $models
     * @return \Illuminate\Notifications\DatabaseNotificationCollection
     */
    public function newCollection(array $models = [])
    {
        return new DatabaseNotificationCollection($models);
    }
}

```

```php
<?php

namespace App\Overrides\Notifications;

trait HasDatabaseNotifications
{
    /**
     * Get the entity's notifications.
     *
     * @return \Illuminate\Database\Eloquent\Relations\MorphMany
     */
    public function notifications()
    {
        return $this->morphMany(DatabaseNotification::class, 'notifiable')->orderBy('created_at', 'desc');
    }

    /**
     * Get the entity's read notifications.
     *
     * @return \Illuminate\Database\Query\Builder
     */
    public function readNotifications()
    {
        return $this->notifications()->whereNotNull('read_at');
    }

    /**
     * Get the entity's unread notifications.
     *
     * @return \Illuminate\Database\Query\Builder
     */
    public function unreadNotifications()
    {
        return $this->notifications()->whereNull('read_at');
    }
}

```

```php
<?php

namespace App\Overrides\Notifications;

use Illuminate\Notifications\RoutesNotifications;

trait Notifiable
{
    use HasDatabaseNotifications, RoutesNotifications;
}

```



## 最后🤪

在你的user类中使用  `App\Overrides\Notifications\Notifiable`  这个  `trait` ，其实就是把 `DatabaseNotification` 的连接改成 `mongodb`，再调用，easy 💩