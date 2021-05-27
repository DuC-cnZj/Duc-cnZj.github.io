---
title: Jsonapi 在 laravel 中简单的实现
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - laravel
tags:
 - Jsonapi
publish: true
---


> [ JSON：API](https://jsonapi.org/format/1.1/)是客户端应如何请求获取或修改资源以及服务器应如何响应这些请求的规范。
>
> 本文为实践教程

为什么不用 laravel 自带的 `API Resources`？

- 官方自带的我觉得很好，我现在也一直用官方自带的，如果你不想用官方的，这个包会是很好的选择
- fractal 比官方自带的包更加符合 jsonapi 的实现和返回规范
- 在将来，我肯定会转向 GraphQL，这是大势所趋 <https://developer.github.com/v4/>



## 🚀 安装 [fractal](https://fractal.thephpleague.com/)

> Fractal为复杂的数据输出提供了一个表示和转换层，就像在RESTful API中找到的那样，并且与JSON非常相似。可以将其视为JSON / YAML /等的视图层。

```bash
composer require league/fractal
```



## 实践

先定义一个 `CustomSerializer`  类，这里我只做了一步，就是把关系数据放到 `relationships`中，更加符合 jsonapi 规范

```php
<?php

namespace App\Serializers;

use League\Fractal\Serializer\ArraySerializer;

class CustomSerializer extends ArraySerializer
{
    public function mergeIncludes($transformedData, $includedData)
    {
        if (! $this->sideloadIncludes()) {
            return array_merge($transformedData, ['relationships' => $includedData]);
        }

        return $transformedData;
    }
}

# 然后在 ServiceProvier 中绑定
public function register()
{
    $this->app->bind(Manager::class, function () {
        $manager = new Manager();
        $manager->setSerializer(new CustomSerializer());

        return $manager;
    });
}
```

创建一个 `Message` 模型，并且和 `User` 建立一对多的关系

```php
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;

class Message extends Model
{
    protected $guarded = [];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}


<?php

namespace App;

use Illuminate\Notifications\Notifiable;
use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
	...

    public function messages () {
        return $this->hasMany(Message::class);
    }
}

```

定义 transformer

```php
<?php

namespace App\Http\Transformers;

use App\User;
use League\Fractal\TransformerAbstract;

class UserTransformer extends TransformerAbstract
{
    protected $availableIncludes = [
        'messages',
    ];

    //    protected $defaultIncludes = [
    //        'messages',
    //    ];

    public function transform(User $user)
    {
        return [
            'id'         => $user->id,
            'email'      => $user->email,
            'name'       => $user->name,
            'created_at' => $user->created_at->diffForHumans(),
            'updated_at' => $user->updated_at->diffForHumans(),
        ];
    }

    public function includeMessages(User $user)
    {
        $messages = $user->messages;

        return $this->collection($messages, new MessageTransformer, 'messages');
    }
}


<?php

namespace App\Http\Transformers;

use App\Message;
use League\Fractal\TransformerAbstract;

class MessageTransformer extends TransformerAbstract
{
    public function transform(Message $message)
    {
        return  [
            'id'         => $message->id,
            'content'    => $message->content,
            'user_id'    => $message->user_id,
            'created_at' => $message->created_at->diffForHumans(),
            'updated_at' => $message->updated_at->diffForHumans(),
        ];
    }
}

```

路由

```php
// http://localhost:8001/?includes=messages&fields[user]=id,name,email,messages&fields[messages]=content,id&excludes=messages
Route::get('/', function (Request $request) {
    $manager = app(Manager::class);
    $manager->parseIncludes($request->input('includes', []));
    $manager->parseExcludes($request->input('excludes', []));
    $manager->parseFieldsets($request->input('fields', []));

    $users = User::with('messages')->paginate(10);
    $resource = new Collection($users, new UserTransformer(), 'user');
    $resource->setPaginator(new IlluminatePaginatorAdapter($users));

    return $manager->createData($resource)->toArray();
});
```

## 最后

> 你就可以根据不同的需求返回不同的字段了

用户参数：`fields[user]=name,email`

```json
// 20190401114806
// http://localhost:8001/?fields[user]=name,email

{
  "user": [
    {
      "email": "darian.labadie@example.org",
      "name": "Mr. Charley Stehr Sr."
    },
    {
      "email": "kyle05@example.com",
      "name": "Zetta Koelpin"
    }
  ],
  "meta": {
    "pagination": {
      "total": 31,
      "count": 2,
      "per_page": 2,
      "current_page": 1,
      "total_pages": 16,
      "links": {
        "next": "http://localhost:8001?page=2"
      }
    }
  }
}
```

用户参数 `includes=messages&fields[user]=name,email,relationships`

```php
// 20190401114832
// http://localhost:8001/?includes=messages&fields[user]=name,email,relationships

{
  "user": [
    {
      "email": "darian.labadie@example.org",
      "name": "Mr. Charley Stehr Sr.",
      "relationships": {
        "messages": {
          "data": [
            {
              "id": 1,
              "content": "Dolore et voluptas nam voluptates consequatur deserunt quia facere.",
              "user_id": 1,
              "created_at": "18 hours ago",
              "updated_at": "18 hours ago"
            },
            {
              "id": 2,
              "content": "Aut ex aliquam veritatis voluptatem ipsum porro.",
              "user_id": 1,
              "created_at": "18 hours ago",
              "updated_at": "18 hours ago"
            },
            {
              "id": 3,
              "content": "Beatae tempore minima corporis in quidem.",
              "user_id": 1,
              "created_at": "18 hours ago",
              "updated_at": "18 hours ago"
            }
          ]
        }
      }
    },
    {
      "email": "kyle05@example.com",
      "name": "Zetta Koelpin",
      "relationships": {
        "messages": {
          "data": [
            
          ]
        }
      }
    }
  ],
  "meta": {
    "pagination": {
      "total": 31,
      "count": 2,
      "per_page": 2,
      "current_page": 1,
      "total_pages": 16,
      "links": {
        "next": "http://localhost:8001?page=2"
      }
    }
  }
}
```

用户参数： `includes=messages&fields[user]=name,email,relationships,created_at&fields[messages]=content,id`

```json
// 20190401114740
// http://localhost:8001/?includes=messages&fields[user]=name,email,relationships,created_at&fields[messages]=content,id

{
  "user": [
    {
      "email": "darian.labadie@example.org",
      "name": "Mr. Charley Stehr Sr.",
      "created_at": "18 hours ago",
      "relationships": {
        "messages": {
          "data": [
            {
              "id": 1,
              "content": "Dolore et voluptas nam voluptates consequatur deserunt quia facere."
            },
            {
              "id": 2,
              "content": "Aut ex aliquam veritatis voluptatem ipsum porro."
            },
            {
              "id": 3,
              "content": "Beatae tempore minima corporis in quidem."
            }
          ]
        }
      }
    },
    {
      "email": "kyle05@example.com",
      "name": "Zetta Koelpin",
      "created_at": "17 hours ago",
      "relationships": {
        "messages": {
          "data": [
            
          ]
        }
      }
    }
  ],
  "meta": {
    "pagination": {
      "total": 31,
      "count": 2,
      "per_page": 2,
      "current_page": 1,
      "total_pages": 16,
      "links": {
        "next": "http://localhost:8001?page=2"
      }
    }
  }
}
```

## ⁉️ 思考 Q&A？

- 是否可以让在 `query` 层面就限制搜索字段，减轻数据库压力，不过这样的话就感觉像是前端直接给我`sql`，我后端执行并返回结果，或者做一层 map 映射数据库中的字段？

