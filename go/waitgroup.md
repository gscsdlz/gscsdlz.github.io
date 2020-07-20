# Go WaitGroup原理

> waitgroup 用于等待多个协程结束，提供Add和Wait两个方法<br/>
> 基于go1.14的源代码，删除了不影响逻辑的代码和注释，需要提前了解Mutex的知识

## 结构

```go
type WaitGroup struct {
    noCopy noCopy
    state1 [3]uint32
}
```

- noCopy是从编译层面确保整个结构体不被复制
- state1 存储了2个参数，一个是计数另一个是信号量

## 简单使用
```go
func main() {
    val := int32(0)
    wg := sync.WaitGroup{}
    wg.Add(1000)
    for i := 0; i < 1000; i++ {
        go func(idx int) {
            atomic.AddInt32(&val, 1)
            wg.Done()
        }(i)
    }
    wg.Wait()
    fmt.Println(val)
}
```

main将等待所有的协程累加完成，并输出1000后退出<br/>等待协程结束如果使用time.Sleep完成的话，是一种不靠谱的办法

## state()

```go
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
    } else {
        return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
    }
}
```

### 32bit的机器上，state1的内容是
低地址 ------------------- 高地址<br/>
信号量 semap ｜ waiter ｜ counter
### 64bit的机器上，state1的内容是
waiter ｜ counter ｜ 信号量 semap

- counter 高位 即需要被等待的协程数量，Add控制
- waiter 低位 处于等待中的协程数量，Wait控制

## Add()

```go
func (wg *WaitGroup) Add(delta int) {
    statep, semap := wg.state()
    state := atomic.AddUint64(statep, uint64(delta)<<32)
    v := int32(state >> 32)
    w := uint32(state)

    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    if v > 0 || w == 0 {
        return
    }
    if *statep != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    *statep = 0
    for ; w != 0; w-- {
        runtime_Semrelease(semap, false, 0)
    }
}
```

我们把三个panic暂时不要去考虑（假设使用waitgroup的调用者不会犯错误），那么add可以进一步简化

即

```go
func (wg *WaitGroup) Add(delta int) {
    statep, semap := wg.state()
    state := atomic.AddUint64(statep, uint64(delta)<<32)
    v := int32(state >> 32)
    w := uint32(state)

    if v > 0 || w == 0 {
        return
    }
    *statep = 0
    for ; w != 0; w-- {
        runtime_Semrelease(semap, false, 0)
    }
}
```

`Done就是一种特殊的Add，Add(-1)`

### 基本流程

根据wg.state的逻辑，statep的高32位为目前需要`被`等待的协程数量，低32位目前调用了wait处于等待状态的协程（被阻塞）

add方法首先将需要被等待的协程数量增加delta个，那么v,w则对应上文的高位和地位

Add立即return的条件
- v > 0 还有至少一个协程没有运行完
- w == 0 没有协程被阻塞（没有协程还在等待wait返回）

反过来说，只有所有被等待的协程完成了且有协程被阻塞，才会通过信号量去唤醒所有等待的协程

w记录了处于等待状态中的协程数量，所以循环w次，来唤醒这些协程。

\*statep = 0则将v,w都清零，`这里无需使用原子方式`，因为正常流程下只有最后一个调用Done的协程有机会执行该操作，所以无需原子。

### 三个panic

我们刚刚分析代码的时候，忽略了三个panic，我们现在来看看这三个panic什么时候会触发

1. 负数的counter<br/>
counter即v，表示需要被等待的协程数量，我们通过Add来添加需要被等待的协程数量，通过Add(-x)或者Done(Add(-1))，如果v小于0，那么说明代码一定有逻辑问题，正常的流程中不可能出现负数。

2. 上一次Wait还没有返回，新的Add被调用<br/>
假设我们在调用Done的时候，已经出现0了且有协程处于等待中，正当我们准备唤醒协程的时候，其他并发的协程再次Add，就会引发第二个panic
 - w != 0所有有协程被阻塞
 - delta > 0表示需要增加
 - v == delta表示增加前v就是0（没有协程需要被等待）

3. 同上<br/>
\*staetp != state 正常情况下，只会有一个协程执行到这里，即最后一个调用Done的，那么这时候不应该还有其他的协程调用Add或者Done（改变高位），也不应该有其他的Wait调用（改变低位）

## Wait()

```go
func (wg *WaitGroup) Wait() {
    statep, semap := wg.state()
    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32)
        if v == 0 {
            return
        }
        // Increment waiters count.
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            runtime_Semacquire(semap)
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            return
        }
    }
}
```

### 基本流程

调用Wait时，如果发现当前没有需要被等待的协程，会立即返回，否则增加等待者数量

通过CAS增加以后会通过信号量阻塞当前协程直到Add上开始唤醒

#### \*statep != 0
当前协程被唤醒的时候，正常流程上，Add上一定是调用了\*statep = 0，如果被唤醒的协程发现这个值不为0，说明waitgroup被错误的调用。

## 总结
1. Add和Done对waitGroup的调用一定要统一，Add(x)就一定要保证Add(-x)
2. Wait可以出现在多个地方
3. Wait还没有返回时，不要调用Add或者Done
4.

## Why? state1
为什么不使用三个原子变量，反而是合并成一个12字节的数组？
