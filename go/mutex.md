# Go Mutex原理

> mutex 互斥锁是golang中很多同步功能的基础，例如RWMutex，syncMap，pool<br/>
> 基于go1.14的源代码，删除了不影响逻辑的代码和注释，用到了MPG模型，需要提前阅读Go调度相关的知识（简单的即可）

- [概述](#概述)
	- [加锁过程](#加锁过程)
	- [解锁过程](#解锁过程)
- [基本结构](#基本结构)
	- [常量](#常量)
- [Lock](#Lock)
	- [自旋检查](#自旋检查)
		- [自旋为什么要检查次数？](#自旋为什么要检查次数？)
		- [为什么单核CPU和运行中的P过少，不能自旋？](#为什么单核CPU和运行中的P过少，不能自旋？)
		- [为什么当前P的队列需要是空的？](#为什么当前P的队列需要是空的？)
		- [抢锁](#抢锁)
- [Unlock](#Unlock)
	- [解锁](#解锁)

## 概述
### 加锁过程
- 首先通过CAS快速设置锁定状态，可以避免仅一个协程时也需要大量的逻辑判断
- 设置状态失败的协程，在满足条件的情况下进行一定的自旋，然后再次尝试获取锁
- 自旋多次还是不能获取到锁，通过信号量（PV原语）将协程阻塞
- 超过1ms不能获取到锁，进入饥饿模式
- `正常模式` 重新尝试获取锁的协程，必须参与排队，和新到的协程共同竞争
- `饥饿模式` 重新尝试获取锁的协程，排在队列头，新到的协程强制排队
- 如果获取到锁的协程发现目前的等待队列只有一个，或者锁的时间不超过1ms，退出饥饿模式

### 解锁过程

## 基本结构

```golang
type Mutex struct {
	state int32  //锁的状态
	sema  uint32 //信号量
}
```

### 常量

```go
const (
	mutexLocked   = 1 //0000 0001 锁定中
	mutexWoken    = 2 //0000 0010
	mutexStarving = 4 //0000 0100 饥饿模式中
)
```

## Lock

加锁时，会通过原子操作判断当前的锁是否已经锁上，如果已经被某个协程锁上了，会进入`lockSlow`逻辑，做自旋、等待等操作

因为是原子操作，假设在初始状态，有两个协程同时执行这个if判断，那么必定只能有一个协程读到的state为0，即未上锁状态，另一个协程读到的值一定不是0（state可能会在lockSlow中被改变）。

```go
func (m *Mutex) Lock() {
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	m.lockSlow()
}
```

### 自旋检查

```go
//检查是否可以自旋
//proc.go@5310
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
		 //已经自旋4次
	if i >= active_spin ||
	   //单核CPU
		 ncpu <= 1 ||  
		 //空闲的P + 正在自旋的P + 1 即没有其他正在运行的P
		 //P的总个数等于 = 空闲 + 自旋 + 非自旋
		 //如果空闲 + 自旋 + 非自旋 >= 总数，意味着非自旋的至多一个了（即当前的P）
		 gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	//返回当前的G所在的P，检查P的本地队列是不是空的，如果是非空的，也不能自旋
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}
```

#### 自旋为什么要检查次数？
- 自旋不能过多，小于4次
- 自旋使用了PAUSE，这个指令会让CPU处于运行状态且不会被硬件层面优化，目前是30次PAUSE
- 如果次数过多，会导致当前G浪费过多的CPU资源
- 但是如果太少了，锁还来不及被其他协程释放，等待锁的协程会过多的被转入休眠（阻塞）

#### 为什么单核CPU和运行中的P过少，不能自旋？
- 单核CPU和其他运行中P的检查，目的是为了避免整个程序没有任何可执行的指令（唯一一个正在运行的P做了自旋PAUSE指令）
- 还有就是当前G所等待的锁，也不会被其他G释放（没有机会）

#### 为什么当前P的队列需要是空的？
- 假设当前检查是否自旋的是G1，持有锁的是G2
- G2持锁后恰好被调度出去，处于当前P的等待队列中
- 如果G1执行了自旋，这时候G2仍处于等待队列中，不会被执行，G1会再次自旋
- 如果G2没有被当前P执行的机会，G1会一直自旋，直到触发上面的检查被阻塞
- 这样的自旋是没有意义的
- G2如果处于其他P上，那么G2的解锁过程和G1的自旋可以同时进行，那么G1自旋完，才有可能获取锁

### 抢锁

```go
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false //是否处于饥饿模式
	awoke := false
	iter := 0 //自旋执行次数
	old := m.state //避免被其他协程改写
	for {
		//处于加锁状态并且不处于饥饿模式，才会尝试判断是否可以自旋
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {

			//awoke为true表示当前协程是刚刚被唤醒的
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			//自旋，汇编指令，PAUSE30次
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}

		new := old
		//当前协程没有处于饥饿模式
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		//锁定中或者饥饿模式中
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}

		//当前处于饥饿模式中，并且锁定中
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
		//如果旧的状态和全局状态一致，则尝试将新的状态写入
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break
			}
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			//处于饥饿模式，立即获取锁，并且返回
			if old&mutexStarving != 0 {
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					//等待时间不到1ms，或者等待锁的协程就一个了，退出饥饿模式
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			//正常模式下，重新排队和新来的协程一起竞争
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}
}
```

## Unlock

```go
func (m *Mutex) Unlock() {
	//取消锁定状态，如果加锁的时候没有协程竞争，那么m.state就等于1，无需唤醒等待的协程
	new := atomic.AddInt32(&m.state, -mutexLocked)
	//new!=0则m.state不等于1，即有其他协程参数竞争【当然也有可能是解锁以后再次解锁】
	if new != 0 {
		m.unlockSlow(new)
	}
}
```

### 解锁
```go
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```
