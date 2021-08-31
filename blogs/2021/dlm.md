---
title: 分布式锁的集中实现
date: '2021-08-31 12:34:11'
sidebar: true
categories:
 - 技术
tags:
 - 分布式锁
publish: true
---

## 无锁代码

### sql

```sql
CREATE TABLE `books` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  `count` int NOT NULL,
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

### 代码实现

```go
package main

import (
	"database/sql"
	"fmt"
	"log"

	_ "github.com/go-sql-driver/mysql"
)

type Book struct {
	ID    int
	Name  string
	Count int
}

var db *sql.DB

func main() {
	var err error
	db, err = sql.Open("mysql", "app:12345@tcp(127.0.0.1:3306)/app")

	if err != nil {
		log.Fatal("数据库连接建立失败")
	}
	defer db.Close()

	Sell(1, 1)
	rows, err := db.Query("SELECT id,name,count FROM books")
	PanicError(err)
	for rows.Next() {
		book := &Book{}
		rows.Scan(&book.ID, &book.Name, &book.Count)
		log.Println(book)
	}
}

func Sell(id, count int) error {
	book := &Book{}
	row := db.QueryRow("SELECT `id`,`name`,`count` FROM books where `id` = ?", id)
	err := row.Scan(&book.ID, &book.Name, &book.Count)
	if err != nil {
		log.Printf("书籍 id = %d 未找到 %v \n", id, err)
		return err
	}
	if count > book.Count {
		return fmt.Errorf("库存不够了，还剩 %d", book.Count)
	}
	res, err := db.Exec("Update books set count = ? where id = ?", book.Count-count, book.ID)
	if err != nil {
		log.Printf("书籍 id = %d 减库存失败 %v\n", id, err)
		return err
	}
	a, _ := res.RowsAffected()
	log.Printf("res.RowsAffected() %v", a)
	return nil
}

func PanicError(e error) {
	if e != nil {
		panic(e)
	}
}
```



### 测试代码

```go
func TestSell(t *testing.T) {
	wg := &sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			if Sell(1, 1) == nil {
				log.Println("库存扣减成功")
			} else {
				log.Println("库存扣减失败")
			}
		}()
	}

	wg.Wait()
}
```



## mysql 悲观锁

使用 `for update` 注意使用其他字段时需要加索引，不然行锁会升级为表锁

```go
func Sell(id, count int) error {
	book := &Book{}
	tx, err := db.Begin()
	if err != nil {
		log.Printf("事务开启失败 %v \n", err)
		return err
	}
	row := tx.QueryRow("SELECT `id`,`name`,`count` FROM books where `id` = ? for update", id)
	err = row.Scan(&book.ID, &book.Name, &book.Count)
	if err != nil {
		log.Printf("书籍 id = %d 未找到 %v \n", id, err)
		tx.Rollback()
		return err
	}
	if count > book.Count {
		tx.Rollback()
		return fmt.Errorf("库存不够了，还剩 %d", book.Count)
	}
	res, err := tx.Exec("Update books set count = ? where id = ?", book.Count-count, book.ID)
	if err != nil {
		tx.Rollback()
		log.Printf("书籍 id = %d 减库存失败 %v\n", id, err)
		return err
	}
	a, _ := res.RowsAffected()
	log.Printf("res.RowsAffected() %v", a)
	tx.Commit()
	return nil
}
```



## mysql乐观锁

>  使用 version 来进行版本控制

```sql
ALTER TABLE `books` ADD COLUMN `version` int NOT NULL DEFAULT '0' COMMENT '';
```

```go
func Sell(id, count int) error {
	book := &Book{}
	var success bool
	for !success {
		row := db.QueryRow("SELECT `id`,`name`,`count`, `version` FROM books where `id` = ?", id)
		err := row.Scan(&book.ID, &book.Name, &book.Count, &book.Version)
		if err != nil {
			log.Printf("书籍 id = %d 未找到 %v \n", id, err)
			return err
		}
		if count > book.Count {
			return fmt.Errorf("库存不够了，还剩 %d", book.Count)
		}
		res, err := db.Exec("Update books set count = ?, version = version+1 where id = ? and version = ?", book.Count-count, book.ID, book.Version)
		if err != nil {
			log.Printf("书籍 id = %d 减库存失败 %v\n", id, err)
			return err
		}
		a, _ := res.RowsAffected()
		if a == 1 {
			success = true
		}
		log.Printf("res.RowsAffected() %v", a)
	}

	return nil
}
```

## redis

#### lock

简单实现，实际情况还需要增加owner，以及考虑redis redlock 问题，参考[redlock实现](https://github.com/go-redsync/redsync)

```go
package main

import (
	"context"
	"log"
	"time"

	"github.com/go-redis/redis/v8"
)

var rdb *redis.Client

func init() {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})
}

type Locker interface {
	Acquire(key string) bool
	Release(key string) bool
}

type RdbLocker struct {
	rdb *redis.Client
}

func NewRdbLocker(rdb *redis.Client) Locker {
	return &RdbLocker{rdb: rdb}
}

func (l *RdbLocker) Acquire(key string) bool {
	r := l.rdb.SetNX(context.TODO(), key, "", time.Second*20)
	log.Println(r.Result())
	rr, _ := r.Result()
	return rr
}

func (l *RdbLocker) Release(key string) bool {
	eval := l.rdb.Eval(context.Background(), `
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
`, []string{key}, "")

	b, _ := eval.Bool()
	return b
}

```

#### sell

```go
func Sell(id, count int) error {
	l := NewRdbLocker(rdb)
	book := &Book{}
	key := fmt.Sprintf("id-%d", id)
	var success bool
	for !success {
		if l.Acquire(key) {
			row := db.QueryRow("SELECT `id`,`name`,`count` FROM books where `id` = ?", id)
			err := row.Scan(&book.ID, &book.Name, &book.Count)
			if err != nil {
				log.Printf("书籍 id = %d 未找到 %v \n", id, err)
				l.Release(key)

				return err
			}
			if count > book.Count {
				l.Release(key)
				return fmt.Errorf("库存不够了，还剩 %d", book.Count)
			}
			res, err := db.Exec("Update books set count = ? where id = ?", book.Count-count, book.ID)
			if err != nil {
				log.Printf("书籍 id = %d 减库存失败 %v\n", id, err)
				l.Release(key)
				return err
			}
			a, _ := res.RowsAffected()
			log.Printf("res.RowsAffected() %v", a)
			l.Release(key)
			success = true
		}
	}

	return nil
}
```





```
 docker run \
  -p 12379:2379 \
  -p 12380:2380 \
  --name etcd-gcr-v3.5.0 \
  gcr.io/etcd-development/etcd:v3.5.0 \
  /usr/local/bin/etcd \
  --name s1 \
  --data-dir /etcd-data \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://0.0.0.0:2379 \
  --listen-peer-urls http://0.0.0.0:2380 \
  --initial-advertise-peer-urls http://0.0.0.0:2380 \
  --initial-cluster s1=http://0.0.0.0:2380 \
  --initial-cluster-token tkn \
  --initial-cluster-state new \
  --log-level info \
  --logger zap \
  --log-outputs stderr
```



## etcd

> etcd 自带 "go.etcd.io/etcd/client/v3/concurrency" 这个包可以实现分布式锁，里面主要使用了租约，详细之后再看

```go
func Sell(id, count int) error {
	s, _ := concurrency.NewSession(client)
	mu := concurrency.NewMutex(s, fmt.Sprintf("/id-%d", id))
	var success bool
	book := &Book{}
	for !success {
		if err := mu.Lock(context.TODO()); err == nil {
			log.Println("here")
			row := db.QueryRow("SELECT `id`,`name`,`count` FROM books where `id` = ?", id)
			err := row.Scan(&book.ID, &book.Name, &book.Count)
			if err != nil {
				mu.Unlock(context.TODO())
				log.Printf("书籍 id = %d 未找到 %v \n", id, err)
				return err
			}
			if count > book.Count {
				mu.Unlock(context.TODO())
				return fmt.Errorf("库存不够了，还剩 %d", book.Count)
			}
			res, err := db.Exec("Update books set count = ? where id = ?", book.Count-count, book.ID)
			if err != nil {
				mu.Unlock(context.TODO())
				log.Printf("书籍 id = %d 减库存失败 %v\n", id, err)
				return err
			}
			a, _ := res.RowsAffected()
			log.Printf("res.RowsAffected() %v", a)
			mu.Unlock(context.TODO())
			success = true
		}
	}
	return nil
}

```




