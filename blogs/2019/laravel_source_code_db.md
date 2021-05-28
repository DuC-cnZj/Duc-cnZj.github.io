---
title: laravel 源码解析之 DB
date: "2019-02-28 08:16:14"
sidebar: false
categories:
 - 技术
tags:
 - laravel
 - 源码解析
 - DB
publish: true
---


`DatabaseServiceProvider.php`随着 `$this->bootstrap();`的 `\Illuminate\Foundation\Bootstrap\RegisterProviders::class,`  启动。

```php
public function register()
{
    Model::clearBootedModels();

    $this->registerConnectionServices();

    $this->registerEloquentFactory();

    $this->registerQueueableEntityResolver();
}

# 首先进入 DatabaseServiceProvider -> register 方法
# 它清除了自身的 $booted，和 $globalScopes，目前还不知道这两个属性做啥，先往下看
public static function clearBootedModels()
{
    static::$booted = [];

    static::$globalScopes = [];
}

# 第二步绑定了键 db.factory、db、db.connection 的实例的实例化闭包，这里未实例化具体的类
protected function registerConnectionServices()
{
    $this->app->singleton('db.factory', function ($app) {
        return new ConnectionFactory($app);
    });

    $this->app->singleton('db', function ($app) {
        return new DatabaseManager($app, $app['db.factory']);
    });

    $this->app->bind('db.connection', function ($app) {
        return $app['db']->connection();
    });
}


# 第三步
# 绑定了 Faker 实例的闭包，可以看到这里去app.php文件里面去取 faker_locale 值
# 绑定了 Illuminate\Database\Eloquent\Factory 的闭包，用到了上面的 Faker 实例
protected function registerEloquentFactory()
{
    $this->app->singleton(FakerGenerator::class, function ($app) {
        return FakerFactory::create($app['config']->get('app.faker_locale', 'en_US'));
    });

    $this->app->singleton(EloquentFactory::class, function ($app) {
        return EloquentFactory::construct(
            $app->make(FakerGenerator::class), $this->app->databasePath('factories')
        );
    });
}

# 第四步
# 注册了队列解析的闭包？有啥用？
protected function registerQueueableEntityResolver()
{
    $this->app->singleton(EntityResolver::class, function () {
        return new QueueEntityResolver;
    });
}

# register 阶段完成

```

下面进入  `$this->bootstrap();`的 `\Illuminate\Foundation\Bootstrap\BootProviders::class,`阶段，也就是执行各个服务的`boot` 方法，

```php
public function boot()
{
    Model::setConnectionResolver($this->app['db']);

    Model::setEventDispatcher($this->app['events']);
}


# $this->app['db']实例化db
# 实例化的过程中需要 db.factory 所以先实例化 db.factory
$this->app->singleton('db', function ($app) {
    return new DatabaseManager($app, $app['db.factory']);
});

$this->app->singleton('db.factory', function ($app) {
    return new ConnectionFactory($app);
});


# 上面这两不为 Model 类添加了$resolver(DatabaseManager实例)和$dispatcher(events实例)

```

**此时还没有连接过数据库！**启动过程中没有连接数据库！



然后我在代码中写了

```php
\DB::table('users')->all();
```

所以它会自动加载Facade，最后走到

这里开始和数据库打交道了

```php
public function __call($method, $parameters)
{
    # $this->connection() 实例化 MySqlConnection 类(但未连接数据库，只是生成闭包)，然后给到 DatabaseManager 中的 connections['mysql'] 属性，读写分离：MySqlConnection有两个属性 $pdo 和 $readPdo,
#    readPdo 设置了 就会去读，默认就是写
    return $this->connection()->$method(...$parameters);
}


public function connection($name = null)
{
    [$database, $type] = $this->parseConnectionName($name);

    $name = $name ?: $database;

	# 如果之前没有创建过(没连接过数据库)数据库，那么意味着还没有db实例，所以需要先连接
    if (! isset($this->connections[$name])) {
        $this->connections[$name] = $this->configure(
            $this->makeConnection($database), $type
        );
    }

    return $this->connections[$name];
}

# 获取数据库默认连接，去拿  'default' => env('DB_CONNECTION', 'mysql'),

protected function parseConnectionName($name)
{
    $name = $name ?: $this->getDefaultConnection();

    # 这里有读写分离设置，文档里也写了怎么配置
    return Str::endsWith($name, ['::read', '::write'])
        ? explode('::', $name, 2) : [$name, null];
}

# 读写分离
'mysql' => [
    'driver' => 'mysql',
    'read' => [
        'host' => ['192.168.1.1'],
    ],
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    'database' => env('DB_DATABASE', 'forge'),
    'username' => env('DB_USERNAME', 'forge'),
    'password' => env('DB_PASSWORD', ''),
    'unix_socket' => env('DB_SOCKET', ''),
    'charset' => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'prefix' => '',
    'prefix_indexes' => true,
    'strict' => true,
    'engine' => null,
],


protected function makeConnection($name)
{
    # 去获取 database.php 里面读数据库配置
    $config = $this->configuration($name);

	# 数据库扩展
    if (isset($this->extensions[$name])) {
        return call_user_func($this->extensions[$name], $config, $name);
    }

    // Next we will check to see if an extension has been registered for a driver
    // and will call the Closure if so, which allows us to have a more generic
    // resolver for the drivers themselves which applies to all connections.
    if (isset($this->extensions[$driver = $config['driver']])) {
        return call_user_func($this->extensions[$driver], $config, $name);
    }

    return $this->factory->make($config, $name);
}


public function make(array $config, $name = null)
{
    # 为数据库添加表前缀
    $config = $this->parseConfig($config, $name);

    # 我特意配了可读权限
    if (isset($config['read'])) {
        return $this->createReadWriteConnection($config);
    }

    return $this->createSingleConnection($config);
}

protected function createReadWriteConnection(array $config)
{
    $connection = $this->createSingleConnection($this->getWriteConfig($config));

    return $connection->setReadPdo($this->createReadPdo($config));
}

protected function getWriteConfig(array $config)
{
    # 这里不配置 write 会报错
    return $this->mergeReadWriteConfig(
        $config, $this->getReadWriteConfig($config, 'write')
    );
}

protected function getReadWriteConfig(array $config, $type)
{
    # 这里不配置 write 会报错 $config['write'][0] 没有 write 就报错
    return isset($config[$type][0])
        ? Arr::random($config[$type])
            : $config[$type];
}

# 回到 createReadWriteConnection 方法
protected function createReadWriteConnection(array $config)
{
    $connection = $this->createSingleConnection($this->getWriteConfig($config));

    return $connection->setReadPdo($this->createReadPdo($config));
}

protected function createSingleConnection(array $config)
{
    $pdo = $this->createPdoResolver($config); # 闭包

    # 返回对应数据库实例
    return $this->createConnection(
        $config['driver'], $pdo, $config['database'], $config['prefix'], $config
    );
}


# 根据 driver(之前解析配置文件的时候解析出来的 driver) 创建对应类型的数据库
protected function createConnection($driver, $connection, $database, $prefix = '', array $config = [])
{
    if ($resolver = Connection::getResolver($driver)) {
        return $resolver($connection, $database, $prefix, $config);
    }

    switch ($driver) {
        case 'mysql':
            return new MySqlConnection($connection, $database, $prefix, $config);
        case 'pgsql':
            return new PostgresConnection($connection, $database, $prefix, $config);
        case 'sqlite':
            return new SQLiteConnection($connection, $database, $prefix, $config);
        case 'sqlsrv':
            return new SqlServerConnection($connection, $database, $prefix, $config);
    }

    throw new InvalidArgumentException("Unsupported driver [{$driver}]");
}

# Connection::getResolver($driver) 这一步走到了 Illuminate\Database\Connection
public static function getResolver($driver)
{
    # 去 Connection 类中看是否解析过mysql连接，注意往下走可以看出 $resolvers 存的是闭包
    return static::$resolvers[$driver] ?? null;
}

# 因为没解析过所以直接，这里可以看出 laravel 支持的连接类型
# return new MySqlConnection($connection, $database, $prefix, $config);
```

接下里进入 `MySqlConnection.php` 这个类继承了 `Illuminate\Database\Connection.php`

```php
public function __construct($pdo, $database = '', $tablePrefix = '', array $config = [])
{
    # 把连接数据库的闭包赋值给 pdo
    $this->pdo = $pdo;

	# 设置数据库名称 
    $this->database = $database;

    # 表前缀
    $this->tablePrefix = $tablePrefix;

    # 所有的config配置
    $this->config = $config;

    # 设置默认的查询语法解析器，注意这个方法被子类 MySqlConnection.php 重写了，这个是mysql的解析器
    # 这个类主要是来拼接 sql 语句，未执行
    $this->useDefaultQueryGrammar();

    # 这个类来执行上面拼接好的 sql 语句
    $this->useDefaultPostProcessor();
}

# MySqlConnection.php
protected function getDefaultQueryGrammar()
{
    return $this->withTablePrefix(new QueryGrammar);
}

# 设置了 Grammar 实例的表前缀
public function withTablePrefix(Grammar $grammar)
{
    $grammar->setTablePrefix($this->tablePrefix);

    return $grammar;
}

# MySqlConnection.php
protected function getDefaultPostProcessor()
{
    return new MySqlProcessor;
}


# 上面的一系列操作都是在 makeConnection 方法中
# 执行完后跳出 makeConnection 方法，进入 configure 方法
public function connection($name = null)
{
    [$database, $type] = $this->parseConnectionName($name);

    $name = $name ?: $database;

	# 如果之前没有创建过(没连接过数据库)数据库，那么意味着还没有db实例，所以需要先连接
    if (! isset($this->connections[$name])) {
        # 把返回的连接实例给到 DatabaseManager 的 connections 属性
        $this->connections[$name] = $this->configure(
            $this->makeConnection($database), $type
        );
    }

    return $this->connections[$name];
}

# configure 方法
protected function configure(Connection $connection, $type)
{
    $connection = $this->setPdoForType($connection, $type);

	# Connection 是否绑定过事件触发器，没有则绑定，这一步把实例的 events 和 $this->app['events'] 进行了绑定
    if ($this->app->bound('events')) {
        $connection->setEventDispatcher($this->app['events']);
    }

    # 设置了 reconnector 闭包
    $connection->setReconnector(function ($connection) {
        # 设置闭包还不会进入里面执行方法
        # 这里实际上是获取了 $connection 连接的 name，也就是 database.php 中的连接名称
        $this->reconnect($connection->getName());
    });

    return $connection;
}

# 根据类型设置 $pdo，读写分离用，如果没分离，那么 $readPdo 和 $pdo 是同一个闭包
# 如果分离了，那么 $readPdo 的闭包将会是读闭包
# $readPdo 和 $pdo 都是 Connection 实例的属性
protected function setPdoForType(Connection $connection, $type = null)
{
    if ($type === 'read') {
        $connection->setPdo($connection->getReadPdo());
    } elseif ($type === 'write') {
        $connection->setReadPdo($connection->getPdo());
    }

    return $connection;
}

# 以上就走完了 $this->connection()，接下来是 ->$method(...$parameters);
# 我在代码中是这样写的 DB::table('users')->get();
# 所以接下来走 Connection 实例的 table 方法
public function __call($method, $parameters)
{
    return $this->connection()->$method(...$parameters);
}


public function table($table)
{
    # $this->query() 返回 Builder 实例
    # 之后的 from get 都是对 Builder 进行操作
    return $this->query()->from($table);
}

# MysqlConnection.php
# use Illuminate\Database\Query\Builder as QueryBuilder;
# 这里实例化了 Builder 类，QueryBuilder 是 Builder 的别名
public function query()
{
    return new QueryBuilder(
        $this,
        $this->getQueryGrammar(),
        $this->getPostProcessor()
    );
}

# get 操作调用了 processor 实例去执行 sql
public function get($columns = ['*'])
{
    # 用 collect 把查询结果包起来，然后返回
    return collect($this->onceWithColumns($columns, function () {
        return $this->processor->processSelect($this, $this->runSelect());
    }));
}

# 不过执行之前，框架做了三个操作
# $this->toSql()
# $this->getBindings()
# 判断 ! $this->useWritePdo，注意这个默认值是 false，取反之后就是 true
protected function runSelect()
{
    return $this->connection->select(
        $this->toSql(), $this->getBindings(), ! $this->useWritePdo
    );
}

# 访问者模式的实践，把自身类传给了别的类去操作
public function toSql()
{
    return $this->grammar->compileSelect($this);
}

# MysqlGrammer.php
public function compileSelect(Builder $query)
{
    if ($query->unions && $query->aggregate) {
        return $this->compileUnionAggregate($query);
    }

    # 这里遍历了mysql所有的操作 where union offset limit...
    $sql = parent::compileSelect($query);

    # 看用户查询是否存在 union 存在的话合并查询语句
    if ($query->unions) {
        $sql = '('.$sql.') '.$this->compileUnions($query);
    }

    return $sql;
}

# parent::compileSelect($query);
# 把用户的查询转换成 sql 语句 "select * from `users`"
public function compileSelect(Builder $query)
{
    if ($query->unions && $query->aggregate) {
        return $this->compileUnionAggregate($query);
    }

    // If the query does not have any columns set, we'll set the columns to the
    // * character to just get all of the columns from the database. Then we
    // can build the query and concatenate all the pieces together as one.
    $original = $query->columns;

    if (is_null($query->columns)) {
        $query->columns = ['*'];
    }

    # 把数组类型的查询转换成 sql 语句 "select * from `users`"
    $sql = trim($this->concatenate(
        # 里面这一步返回了数组
        # $sql => [
        # columns: "select *",
        # from: "",
        # where: ""
        # ]
        $this->compileComponents($query))
               );

    # 设置查询字段
    $query->columns = $original;

    return $sql;
}

# select 
public function select($query, $bindings = [], $useReadPdo = true)
{
    return $this->run($query, $bindings, function ($query, $bindings) use ($useReadPdo) {
        if ($this->pretending()) {
            return [];
        }

		# prepared 方法还触发了 Events\StatementPrepared 事件
        $statement = $this->prepared($this->getPdoForSelect($useReadPdo)
                                     ->prepare($query));

        $this->bindValues($statement, $this->prepareBindings($bindings));

        # 执行
        $statement->execute();

        return $statement->fetchAll();
    });
}

# run
protected function run($query, $bindings, Closure $callback)
{
    # 重新设置 Connection 的实例
    $this->reconnectIfMissingConnection();

    # 统计了 sql 执行时间
    $start = microtime(true);

    try {
        # 这里终于开始调用我们很早就准备好的连接数据库的实例了，也就是 createPdoResolverWithHosts 中的闭包
        $result = $this->runQueryCallback($query, $bindings, $callback);
    } catch (QueryException $e) {
        $result = $this->handleQueryException(
            $e,
            $query,
            $bindings,
            $callback
        );
    }

	#  触发 QueryExecuted 事件
    $this->logQuery(
        $query,
        $bindings,
        $this->getElapsedTime($start)
    );

    return $result;
}

public function logQuery($query, $bindings, $time = null)
{
    $this->event(new QueryExecuted($query, $bindings, $time, $this));

    # 是否要记录查询语句，绑定属性，执行事件，默认不记录，使用 enableQueryLog 开启记录
    if ($this->loggingQueries) {
        $this->queryLog[] = compact('query', 'bindings', 'time');
    }
}

protected function reconnectIfMissingConnection()
{
    if (is_null($this->pdo)) {
        $this->reconnect();
    }
}

public function reconnect()
{
    # 这里就看出上面为什么要设置 reconnector 闭包了
    if (is_callable($this->reconnector)) {
        $this->doctrineConnection = null;

        return call_user_func($this->reconnector, $this);
    }

    throw new LogicException('Lost connection and no reconnector available.');
}


# 这里就可以看出 Connection 实例的 pdo 是写库/默认操作库
protected function getPdoForSelect($useReadPdo = true)
{
    return $useReadPdo ? $this->getReadPdo() : $this->getPdo();
}

protected function createPdoResolverWithHosts(array $config)
{
    return function () use ($config) {
        # 这里用了 shuffle 代表从你配置的 host 数组中随机一个 host 去连接
        foreach (Arr::shuffle($hosts = $this->parseHosts($config)) as $key => $host) {
            $config['host'] = $host;

            try {
                # 调用 MySqlConnector 去连接数据库
                return $this->createConnector($config)->connect($config);
            } catch (PDOException $e) {
                continue;
            }
        }

        throw $e;
    };
}

# 你不配置 host 就抛出异常了
protected function parseHosts(array $config)
{
    $hosts = Arr::wrap($config['host']);

    if (empty($hosts)) {
        throw new InvalidArgumentException('Database hosts array is empty.');
    }

    return $hosts;
}

'mysql' => [
    'read' => [
        'host' => ['192.168.1.1', '127.0.0.1'], // 对应 Arr::shuffle($hosts = $this->parseHosts($config))
    ],
    'write' => [
        'host' => ['196.168.1.2'],
    ],
    'sticky'    => true,
    'driver'    => 'mysql',
    'database'  => 'database',
    'username'  => 'root',
    'password'  => '',
    'charset'   => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'prefix'    => '',
],


public function connect(array $config)
{
    # 解析配置 "mysql:host=db;port=3306;dbname=homestead"
    $dsn = $this->getDsn($config);

    $options = $this->getOptions($config);

    # 真正连接数据库
    $connection = $this->createConnection($dsn, $config, $options);

    # 选库
    if (! empty($config['database'])) {
        $connection->exec("use `{$config['database']}`;");
    }

    # 设置编码方式
    $this->configureEncoding($connection, $config);

    // Next, we will check to see if a timezone has been specified in this config
    // and if it has we will issue a statement to modify the timezone with the
    // database. Setting this DB timezone is an optional configuration item.
    $this->configureTimezone($connection, $config);

    # 设置模式，严格模式是个什么东西？回头去看看
    $this->setModes($connection, $config);

    return $connection;
}
```

当你读写分离的时候 ，使用readPdo，框架内部有一个$useReadPdo变量，专门在你读的时候发挥作用，它会在你配置了读数据库的时候去使用读库，如果没配置，则去使用默认pdo也就是写库。

只配读库不配写库会报错

```php
# 这里报错，extract 把数组里面的键值拿出来当变量了，不配的话 Undefined variable: host
protected function getHostDsn(array $config)
{
    extract($config, EXTR_SKIP);

    return isset($port)
        ? "mysql:host={$host};port={$port};dbname={$database}"
        : "mysql:host={$host};dbname={$database}";
}
```

## Eloquent Model

所有的模型都继承了 `Illuminate\Database\Eloquent\Model`，在使用不存在的静态方法的时候会走 `__callStatic` 方法

```php
public static function __callStatic($method, $parameters)
{
    return (new static)->$method(...$parameters);
}
```

实例化了用户模型

```php
public function __construct(array $attributes = [])
{
    # 详细去解释下，请看下文👇
    $this->bootIfNotBooted();

    # 调用所有 traits 中定义的 initialize + trait 类名的方法
    $this->initializeTraits();

    # 给模型设置一个空的 original 属性
    $this->syncOriginal();

    # 给该属性赋值一个空数组
    # $original => []
    $this->fill($attributes);
}
```

其中 `fill` 方法

```php
public function fill(array $attributes)
{
    $totallyGuarded = $this->totallyGuarded();

	# 从 fillable 属性数组中剔除 guard 数组中的属性，未仔细看，有兴趣的自己去深入把
    foreach ($this->fillableFromArray($attributes) as $key => $value) {
        $key = $this->removeTableFromKey($key);

        // The developers may choose to place some attributes in the "fillable" array
        // which means only those attributes may be set through mass assignment to
        // the model, and all others will just get ignored for security reasons.
        if ($this->isFillable($key)) {
            $this->setAttribute($key, $value);
        } elseif ($totallyGuarded) {
            throw new MassAssignmentException(sprintf(
                'Add [%s] to fillable property to allow mass assignment on [%s].',
                $key, get_class($this)
            ));
        }
    }

    return $this;
}

public function totallyGuarded()
{
    return count($this->getFillable()) === 0 && $this->getGuarded() == ['*'];
}
```

实例化完模型后调用用户的方法，比如 `find`，因为用户没有在模型中定义过该方法，并且类中也没有该方法，所以调用 `__call`

```php
public function __call($method, $parameters)
{
    if (in_array($method, ['increment', 'decrement'])) {
        return $this->$method(...$parameters);
    }

    return $this->forwardCallTo($this->newQuery(), $method, $parameters);
}

# 返回一个新的 query Builder
public function newQuery()
{
    return $this->registerGlobalScopes($this->newQueryWithoutScopes());
}

public function newQueryWithoutScopes()
{
    return $this->newModelQuery()
        # 预加载
        ->with($this->with)
        # 返回关联模型的 count
        ->withCount($this->withCount);
}

public function newModelQuery()
{
    # 返回 \Illuminate\Database\Eloquent\Builder 实例
    # setModel 方法👇
    return $this->newEloquentBuilder(
        $this->newBaseQueryBuilder()
    )->setModel($this);
}

# 这一步就和直接使用 DB 一样了，返回一个 Builder 实例
protected function newBaseQueryBuilder()
{
    $connection = $this->getConnection();

    return new QueryBuilder(
        $connection, $connection->getQueryGrammar(), $connection->getPostProcessor()
    );
}

# 设置模型和表名
public function setModel(Model $model)
{
    $this->model = $model;

    $this->query->from($model->getTable());

    return $this;
}

public function getTable()
{
    # 设置表名的时候会先去看模型是否有 table 属性，没有则根据表的复数来设定表名
    if (! isset($this->table)) {
        return str_replace(
            '\\', '', Str::snake(Str::plural(class_basename($this)))
        );
    }

    return $this->table;
}

# 巧妙的抽离，调用 find 方法
protected function forwardCallTo($object, $method, $parameters)
{
    try {
        return $object->{$method}(...$parameters);
    } catch (Error | BadMethodCallException $e) {
        $pattern = '~^Call to undefined method (?P<class>[^:]+)::(?P<method>[^\(]+)\(\)$~';

        if (! preg_match($pattern, $e->getMessage(), $matches)) {
            throw $e;
        }

        if ($matches['class'] != get_class($object) ||
            $matches['method'] != $method) {
            throw $e;
        }

        static::throwBadMethodCallException($method);
    }
}
```

### bootIfNotBooted

> 第一次使用模型的时候，该模型必定是未 `booted` 状态，所以会进入启动程序

```php
protected function bootIfNotBooted()
{
    if (! isset(static::$booted[static::class])) {
        # 设置该模型已经启动
        static::$booted[static::class] = true;

        # 触发模型启动事件
        $this->fireModelEvent('booting', false);

        static::boot();

        # 触发启动成功的事件
        $this->fireModelEvent('booted', false);
    }
}

# 加载模型的 traits
protected static function boot()
{
    static::bootTraits();
}

protected static function bootTraits()
{
    $class = static::class;

    $booted = [];

    static::$traitInitializers[$class] = [];

    # 获取模型所以的 traits 类，并去重
    foreach (class_uses_recursive($class) as $trait) {
        # 获取 trait 的类名，并且拼接上 boot 前缀
        $method = 'boot'.class_basename($trait);

        # 如果存在 boot + 类名的方法并且该方法还未调用过则调用
        if (method_exists($class, $method) && ! in_array($method, $booted)) {
            # 调用模型类的静态方法 forward_static_call 意味着你的 boot + 类名 的方法必须是静态方法
            forward_static_call([$class, $method]);

            $booted[] = $method;
        }

        # 这里可以看出你还可以定义一个 initialize + 类名的方法，但是这个方法只是记录了方法名称，并未调用
        if (method_exists($class, $method = 'initialize'.class_basename($trait))) {
            static::$traitInitializers[$class][] = $method;

            static::$traitInitializers[$class] = array_unique(
                static::$traitInitializers[$class]
            );
        }
    }
}
```

## 注意 $cast 属性

当你的类型定位这些时，框架会 json_decode `'array', 'json', 'object', 'collection'`然后插入数据库，其他修改之后再插入的类型有：`decimal,update_at,created_at`，别的类型比如 boolean，int，这些存的时候存的都是原值，取出来的时候才做了类型转化

```php
protected function castAttribute($key, $value)
{
    if (is_null($value)) {
        return $value;
    }

    switch ($this->getCastType($key)) {
        case 'int':
        case 'integer':
            return (int) $value;
        case 'real':
        case 'float':
        case 'double':
            return $this->fromFloat($value);
        case 'decimal':
            return $this->asDecimal($value, explode(':', $this->getCasts()[$key], 2)[1]);
        case 'string':
            return (string) $value;
        case 'bool':
        case 'boolean':
            return (bool) $value;
        case 'object':
            return $this->fromJson($value, true);
        case 'array':
        case 'json':
            return $this->fromJson($value);
        case 'collection':
            return new BaseCollection($this->fromJson($value));
        case 'date':
            return $this->asDate($value);
        case 'datetime':
        case 'custom_datetime':
            return $this->asDateTime($value);
        case 'timestamp':
            return $this->asTimestamp($value);
        default:
            return $value;
    }
```

