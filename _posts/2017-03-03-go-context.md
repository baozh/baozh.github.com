---
layout: post
title: "Go Context的踩坑经历"
date: 2018-03-03 16:20
comments: true
categories:
tags:
---


golang在1.7 version引入了context包，它用来处理单个请求在多个go routine的传递上下文数据、(手动/超时)中止routine树。本文主要描述context的实现原理、在实践中遇到的坑。

<!--more-->

## 使用

可通过[官方博客](https://blog.golang.org/context)了解context的用法。

## 原理

### 核心接口Context

```go
type Context interface {
	// Deadline returns the time when work done on behalf of this context
	// should be canceled. Deadline returns ok==false when no deadline is
	// set.
	Deadline() (deadline time.Time, ok bool)
	// Done returns a channel that's closed when work done on behalf of this
	// context should be canceled.
	Done() <-chan struct{}
	// Err returns a non-nil error value after Done is closed.
	Err() error
	// Value returns the value associated with this context for key.
	Value(key interface{}) interface{}
}
```

### 上下文数据的存储与查询

```go
type valueCtx struct {
	Context
	key, val interface{}
}

func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	......
	return &valueCtx{parent, key, val}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

context上下文数据的存储就像一个树，每个结点只存储一个key/value对。`WithValue()`保存一个key/value对，它将父context嵌入到新的子context，并在节点中保存了key/value数据。`Value()`查询key对应的value数据，会从当前context中查询，如果查不到，会递归查询父context中的数据。

值得注意的是，context中的上下文数据并不是全局的，它只查询本节点及父节点们的数据，不能查询兄弟节点的数据。

### 手动cancel和超时cancel

`cancelCtx`中嵌入了父Context，实现了canceler接口：

```go
type cancelCtx struct {
	Context      // 保存parent Context
	done chan struct{}
	mu       sync.Mutex
	children map[canceler]struct{}
	err      error
}

// A canceler is a context type that can be canceled directly. The
// implementations are *cancelCtx and *timerCtx.
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}
```

`cancelCtx`结构体中`children`保存它的所有`子canceler`， 当外部触发cancel时，会调用`children`中的所有`cancel()`来终止所有的`cancelCtx`。`done`用来标识是否已被cancel。当外部触发cancel、或者父Context的channel关闭时，此done也会关闭。

```go
type timerCtx struct {
	cancelCtx     //cancelCtx.Done()关闭的时机：1）用户调用cancel 2）deadline到了 3）父Context的done关闭了
	timer    *time.Timer
	deadline time.Time
}

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
	......
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  deadline,
	}
	propagateCancel(parent, c)
	d := time.Until(deadline)
	if d <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(true, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(d, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
`timerCtx`结构体中`deadline`保存了超时的时间，当超过这个时间，会触发`cancel`。

可以看出，`cancelCtx`也是一棵树，当触发`cancel`时，会cancel本结点和其子树的所有`cancelCtx`。

## 遇到的坑

某天，为了支持etrace，需要在gRpc调用过程中传递requestId、rpcId，我们的解决方案是`Context`。上线后，遇到一系列的坑......

### 背景补充

所有mysql、mysql、redis的操作接口前都会加上context，如果这个context(或其父context)被cancel了，则操作会失败。

```go
func (tx *Tx) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
func(process func(context.Context, redis.Cmder) error) func(context.Context, redis.Cmder) error
func (ch *Channel) Consume(ctx context.Context, handler Handler, queue string, dc <-chan amqp.Delivery) error
func (ch *Channel) Publish(ctx context.Context, exchange, key string, mandatory, immediate bool, msg Publishing) (err error)
```

### Case 1

现象：上线后，5分钟后所有用户登录失败，不断收到报警。

原因：程序中使用localCache，会每5分钟Refresh(调用注册的回调函数)一次所缓存的变量。localCache中保存了一个context，在调用回调函数时会传进去。如果回调函数依赖context，可能会产生意外的结果。

程序中，回调函数`getAppIDAndAlias`的功能是从mysql中读取相关数据。如果ctx被canel了，会直接返回失败。
```
func getAppIDAndAlias(ctx context.Context, appKey, appSecret string) (string, string, error)
```

第一次localCache.Get(ctx, appKey, appSeret)传的ctx是gRpc call传进来的context，而gRpc在请求结束或失败时会cancel掉context，导致之后cache Refresh()时，执行失败。

解决方法：在Refresh时不使用localCache的context，使用一个不会cancel的context。

### Case 2

上线后，不断收到报警(sys err过多)。看log/etrace产生2种sys err：

- context canceled
- sql: Transaction has already been committed or rolled back

#### 背景及原因

![](/assets/images/context-ticket-precedure.jpeg)

`ticket`对外提供Http Restful服务，其中包含了一个`grpc-gateway`，用来将restful协议转成grpc协议。复现`context canceled`的流程如下：

- 客户端发送http restful请求
- grpc-gateway与客户端建立连接，接收请求，转换参数，调用后面的grpc-server。
- grpc-server处理请求。其中，grpc-server会对每个请求启一个stream，由这个stream创建context。
- 客户端连接断开
- grpc-gateway收到连接断开的信号，导致context cancel。grpc client在发送rpc请求后由于外部异常使它的请求终止了(即它的context被cancel)，会发一个RST_STREAM。
- grpc server收到后，马上终止请求（即grpc server的stream context被cancel）。

可以看出，是因为gRpc handler在处理过程中连接被断开。

**`sql: Transaction has already been committed or rolled back`产生的原因：**

程序中使用了官方database包来执行db transaction。其中，在db.BeginTx时，会启一个协程awaitDone：

```
func (tx *Tx) awaitDone() {
    // Wait for either the transaction to be committed or rolled
    // back, or for the associated context to be closed.
    <-tx.ctx.Done()

    // Discard and close the connection used to ensure the
    // transaction is closed and the resources are released.  This
    // rollback does nothing if the transaction has already been
    // committed or rolled back.
    tx.rollback(true)
}
```
在context被cancel时，会进行rollback()，而rollback时，会操作原子变量。之后，在另一个协程中tx.Commit()时，会判断原子变量，如果变了，会抛出错误。

#### 解决方法
这两个error都是由连接断开导致的，是正常的。可忽略这两个error。

### Case 3

上线后，每两天左右有1~2次的mysql事务阻塞，导致请求耗时达到120秒。在盘古(内部的mysql运维平台)中查询到所有阻塞的事务在处理同一条记录。

![现象](/assets/images/context-transaction.png){:height="500px"}

#### 处理过程

1. 初步怀疑是跨机房的多个事务操作同一条记录导致的。由于跨机房操作，耗时会增加，导致阻塞了其他机房执行的db事务。
2. 出现此现象时，暂时将某个接口降级。降低多个事务操作同一记录的概率。
3. 减少事务的个数。
    - 将单条sql的事务去掉
    - 通过业务逻辑的转移减少不必要的事务
4. 调整db参数`innodb_lock_wait_timeout`(120s->50s)。这个参数指示mysql在执行事务时阻塞的最大时间，将这个时间减少，来减少整个操作的耗时。考虑过在程序中指定事务的超时时间，但是`innodb_lock_wait_timeout`要么是全局，要么是session的。担心影响到session上的其它sql，所以没设置。
5. 考虑使用分布式锁来减少操作同一条记录的事务的并发量。但由于时间关系，没做这块的改进。
6. DAL同事发现有事务没提交，查看代码，找到root cause。

原因是golang官方包database/sql会在某种竞态条件下，导致事务既没有commit，也没有rollback。

#### 源码描述

开始事务[BeginTxx()](https://github.com/golang/go/blob/master/src/database/sql/sql.go#L1595)时会启一个协程：

```
// awaitDone blocks until the context in Tx is canceled and rolls back
// the transaction if it's not already done.
func (tx *Tx) awaitDone() {
	// Wait for either the transaction to be committed or rolled
	// back, or for the associated context to be closed.
	<-tx.ctx.Done()

	// Discard and close the connection used to ensure the
	// transaction is closed and the resources are released.  This
	// rollback does nothing if the transaction has already been
	// committed or rolled back.
	tx.rollback(true)
}
```
`tx.rollback(true)`中，会先判断原子变量tx.done是否为1，如果1，则返回；如果是0，则加1，并进行rollback操作。

在提交事务Commit()时，会先操作原子变量tx.done，然后判断context是否被cancel了，如果被cancel，则返回；如果没有，则进行commit操作。

```
// Commit commits the transaction.
func (tx *Tx) Commit() error {
	if !atomic.CompareAndSwapInt32(&tx.done, 0, 1) {
		return ErrTxDone
	}

	select {
	default:
	case <-tx.ctx.Done():
		return tx.ctx.Err()
	}
	var err error
	withLock(tx.dc, func() {
		err = tx.txi.Commit()
	})
	if err != driver.ErrBadConn {
		tx.closePrepared()
	}
	tx.close(err)
	return err
}
```

如果先进行commit()过程中，先操作原子变量，然后context被cancel，之后另一个协程在进行rollback()会因为原子变量置为1而返回。导致commit()没有执行，rollback()也没有执行。

#### 解决方法

可以在执行事务时传进去一个不会cancel的context，也可以修正`database/sql`源码，然后在编译时指定新的go编译镜像。我们之后给golang提交了[patch](https://github.com/golang/go/commit/bcf964de5e16486cec2e102c929768778f50eea2)，修正了此问题(已合入go 1.9.3)。

## 经验教训

由于go大量的官方库、第三方库用到了context，所以调用接受context的函数时要小心，要非常清楚context在什么时候cancel，什么行为会触发cancel。笔者也在此后花了挺长时间总结了gRpc、内部基础库中context的生命期及行为，避免出现同样的问题。











