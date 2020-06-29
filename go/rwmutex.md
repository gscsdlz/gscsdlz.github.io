# Go RWMutex原理

> rwmutex 读写锁基于mutex，适用于读多写少的场景，可以获取多个读锁，但是只能有一个协程获取到写锁<br/>
> 基于go1.14的源代码，删除了不影响逻辑的代码和注释，需要提前了解Mutex的知识

- [结构](#结构)
- [简单使用](#简单使用)
- [Lock](#lock)
  - [为什么需要Add减去一个数字又再次加回来？](#为什么需要add减去一个数字又再次加回来？)
  - [为什么不可以直接使用r!=0？](#为什么不可以直接使用r!=0？)
- [Unlock](#unlock)
  - [解写锁时的r代表什么](#解写锁时的r代表什么)
- [RLock](#rlock)
- [RUnlock](#runlock)
  - [为什么需要等readerWait为1时，才唤醒等待的写锁？](#为什么需要等readerwait为1时，才唤醒等待的写锁？)
- [为什么读写锁在读多写少的场景下很快](#为什么读写锁在读多写少的场景下很快)
- [参考图片](#参考图片)

## 结构

```go
type RWMutex struct {
    w           Mutex  //获取写锁时的互斥锁
    writerSem   uint32 //写锁的信号量
    readerSem   uint32 //读锁的信号量
    readerCount int32  //获取到读锁的协程数
    readerWait  int32  //等待获取读锁的协程数
}

const rwmutexMaxReaders = 1 << 30 //最大读锁数量

```

## 简单使用
```go
package main

import (
    "fmt"
    "sync"
)

type ConcurrencyMap struct {
    RWLocker sync.RWMutex
    Data     map[interface{}]interface{}
}

func NewConCurrencyMap() *ConcurrencyMap {
    return &ConcurrencyMap{
        RWLocker: sync.RWMutex{},
        Data:     make(map[interface{}]interface{}),
    }
}

func (m *ConcurrencyMap) Get(key interface{}) (interface{}, error) {
    m.RWLocker.RLock()
    defer m.RWLocker.RUnlock()
    if d, ok := m.Data[key]; !ok {
        return nil, fmt.Errorf("no such key")
    } else {
        return d, nil
    }
}

func (m *ConcurrencyMap) Put(key interface{}, value interface{}) {
    m.RWLocker.Lock()
    m.Data[key] = value
    m.RWLocker.Unlock()
}

func main() {
    m := NewConCurrencyMap()
    wg := sync.WaitGroup{}
    wg.Add(1000)
    for i := 0; i < 1000; i++ {
        go func(idx int) {
            if idx <= 500 {
                m.Put(idx, idx)
            } else {
                fmt.Println(m.Get(idx - 500))
            }
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```

一个简单的可并发读写的map，我们知道原生的map是禁止并发读写的。

## Lock

> 先看写锁再看读锁 容易理解

```go
func (rw *RWMutex) Lock() {
    //抢互斥锁
    rw.w.Lock()

    //将读锁个数变为负数（也可能是0），阻塞之后尝试获取读锁的所有协程
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders

    //如果r为0，则表示readerCount也是0，即没有任何正在使用读锁的协程，无需阻塞当前协程
    //如果r不为0，则表示当前有活动的读，设置当前协程需要等待的读锁个数为r
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```

### 为什么需要Add减去一个数字又再次加回来？
直接使用LoadInt32，会导致写锁准备加锁时，还会不断的有新的读锁请求，这里直接减去最大值，
将读锁个数变为非正数，阻塞新的获取读锁协程

### 为什么不可以直接使用r!=0？
如果直接使用r!=0可能会出现，AddInt32时正在操作的读锁，在写锁准备阻塞自己时，已经完成了，如果这时候写锁被阻塞，那么没有任何读锁能唤醒这个写锁。

配合上readerWait，当读锁解锁时，readerWait-1，如果等待的r个读锁全部完成，那么r会变成-r,配合上-r+r = 0,写锁不需要阻塞自己

## Unlock

```go
func (rw *RWMutex) Unlock() {
    //将readerCount改为非负数，允许新的读锁被获取
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    if r >= rwmutexMaxReaders {
    	throw("sync: Unlock of unlocked RWMutex")
    }
    //唤醒之前被阻塞的全部协程
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    //释放互斥锁
    rw.w.Unlock()
}
```

### 解写锁时的r代表什么
解写锁时的r表示的是，被写锁阻塞的获取读锁的新协程，和加锁获取到的r不是一个值。

## RLock

```go
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}
```

- 通过原子的增加获取到读锁的协程数，如果发现返回值为负数，则阻塞当前协程
- 为什么负数需要阻塞？[见上文](#lock)

## RUnlock

```go
func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        if r+1 == 0 || r+1 == -rwmutexMaxReaders {
            throw("sync: RUnlock of unlocked RWMutex")
        }
        if atomic.AddInt32(&rw.readerWait, -1) == 0 {
            runtime_Semrelease(&rw.writerSem, false, 1)
        }
    }
}
```

通过原子的减少读锁个数，如果个数小于0，则说明当前的readerCount本来就是非正数，三种情况下是非正数的

1. 读锁已经全部解除且没有写锁，这时候readerCount等于0
2. 读锁已经全部解除但是写锁生效中，这时候readerCount等于-rwmutexMaxReaders
3. 写锁将readerCount减少了rwmutexMaxReaders

1、2点对应的就是异常解锁

### 为什么需要等readerWait为1时，才唤醒等待的写锁？
因为如果提前唤醒写锁，会导致其他读锁还没来得及退出，等于1时，代表的就是当前正在解除写锁的协程，即最后一个活跃的读锁。

## 为什么读写锁在读多写少的场景下很快
考虑一种极端情况`没有写`,那么所有读锁的获取和解除仅仅是对一个变量的原子加减而已，没有任何的等待。

反过来，如果在读少写多的情况下错误的使用了读写锁，性能会变差，因为多了很多读写锁的操作。

## 参考图片
![读写锁状态变化图](/resource/image/读写锁状态变化图.svg)
