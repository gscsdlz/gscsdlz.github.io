
# Go Once原理

>  Once是一种用来保证被调用的函数仅运行一次的struct<br/>
基于go1.14的源代码，删除了不影响逻辑的代码和注释。<br/>
Once可能是整个sync包里面，最简单的工具了。。

- [结构](#结构)
- [用法](#用法)
  - [多个协程](#多个协程)
- [Do](#Do)
  - [为什么要互斥锁？](#为什么要互斥锁？)
  - [为什么要二次判断？](#为什么要二次判断？)
  - [为什么不可以直接使用互斥锁？](#为什么不可以直接使用互斥锁？)
  - [互斥锁会影响性能吗？](#互斥锁会影响性能吗？)
- [官方的注释](#官方的注释)
- [错误的实现](#错误的实现)

## 结构
```go
type Once struct {
    done uint32
    m    Mutex
}
```

- m是一个互斥锁
- done是一个标志位，一般只有两种值，0/1
- done=0，表示当前的func还没有执行
- done=1，反之

## 用法

Once只需要两步
- 初始化一个Once结构体
- 调用Do
- 对于同一个Once结构体来说，Do里面的func只会被调用一次，也就是如果每个协程都去初始化自己的Once结构是没有任何意义的。

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    o := sync.Once{}

    o.Do(func() {
        fmt.Println("ok")
    })
}
```

### 多个协程

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    o := sync.Once{}
    go func() {
        o.Do(func() {
            fmt.Println("ok")
        })
    }()
    go func() {
        o.Do(func() {
            fmt.Println("ok")
        })
    }()
    time.Sleep(1 * time.Second)
}
```

两者都只输出一个"ok"


## Do

```go
func (o *Once) Do(f func()) {
  //是不是已经运行过一次了，如果是则不需要操作了
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 { //二次判断
    //将状态设置为已完成
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

我们`假设f()是一个肯定会执行成功，且没有任何异常的函数`，将Do进行一个小的改动（取消defer）

```go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
      o.m.Lock()
        if o.done == 0 {
            f()
            atomic.StoreUint32(&o.done, 1)
        }
      o.m.Unlock()
    }
}
```

### 为什么要互斥锁？
假设取消互斥锁<br/>
G1，G2表示两个Goroutine<br/>
那么当他们在不同的P上并发执行的时候（假设每个时间段执行同一个语句）<br/>
第一个原子检查和第二个二次检查都会成功，f()被执行了两次

### 为什么要二次判断？
假设我们仅保留锁取消二次检查<br/>
同样的G1、G2，同样的执行条件<br/>
两个G同时通过原子检查，G1拿到锁，执行f()，G2等待<br/>
G1执行完成，释放锁，G2会再次执行f()

### 为什么不可以直接使用互斥锁？
如果直接使用互斥锁，会导致那些后来再执行Do操作的协程也要不断的加锁，解锁

### 互斥锁会影响性能吗？
肯定会，但是仅仅最开始多个协程同一时间发起操作是会出现<br/>
一般来说Once用来初始化数据库的连接之类的，应用启动的时候，不太可能出现大面积的阻塞<br/>
但是也不建议，Do里面放一堆耗时操作。

## 官方的注释
> 瞎翻译的

每个Once实例的Do方法，仅在首次执行的时候会去调用f(即参数）。换句话说，对于`var once Once`,如果`once.Do(f)`被调用了很多次，仅第一个调用能执行f，当然每次调用的时候f可以不一样。对于每个需要执行的函数都需要一个新的Once实例

Do一般用于仅能运行一次的初始化操作中，因为f是一个无参函数，所以你需要使用字面量来捕获需要有Do调用的函数的参数，例如`config.once.Do(func() { config.init(filename) })`

在上一个正在执行f的G返回前，新的调用是不会返回的（被Mutex阻塞），所以如果f调用了Do，会引起死锁

例如
```go
o := sync.Once{}
o.Do(func() {
  o.Do(func() {
    fmt.Println("ok")
  })
})
```

`fatal error: all goroutines are asleep - deadlock!`

## 错误的实现

```go
if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	f()
}
```
由于Once.Do()是保证当Do返回时，f已经完成了<br/>
使用CAS的方式，当第一个G正在执行f的时候，第二个G，就直接返回了<br/>
如果这个Once用于初始化连接，那么G2很有可能拿到的就是空连接
