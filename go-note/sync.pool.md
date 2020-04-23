# sync.pool
#### go 1.12

```go
// 整个进程的对象池
type Pool struct {
	noCopy noCopy

	// p对象池的数量总是小于或者等于P的数量
	local     unsafe.Pointer	// [p]poolLocal
	
	localSize uintptr			// 对象池数量 len(local)。
	
	// 当对象池中无可用对象，会通过这个方法去创建一个新的对象。
	New func() interface{}		
}	
```

```go
// p本地的对象池
type poolLocalInternal struct {
	// 	每个对象池都绑定在p上面
	// 每个对象池都有p私有的一个对象
	private interface{}   // Can be used only by the respective P.
	// 一个共享的对象数组，GET和PUT时都会依据此来拓展
	shared  []interface{} // Can be used by any P.
	Mutex                 // Protects shared.
}
```

```go
// 对外提供的接口，向对象池中put
func (p *Pool) Put(x interface{}) {
	if x == nil {return}
	l := p.pin() // 这个函数可以从对象池中获取每个p的本地对象池
	// 如果p本地私有的对象是空的，就直接赋值就好了
	// 这里不需要加锁。因为pin函数会持有一个大锁
	if l.private == nil { 
		l.private = x
		x = nil
	}
	// 释放pin函数内持有的锁
	runtime_procUnpin()
	// 这个地方需要加锁
	// 如果放到p私有，则x为nil。
	if x != nil {
		l.Lock()
		l.shared = append(l.shared, x)
		l.Unlock()
	}
}
```

```go
// 这个函数没什么好说的
// 传入一个起点地址，传入一个索引
// 根据每个p的本地pool的的大小得到具体的pool
// l就是进程全局对象池的local
// i就是p的编号
// go程序在初始化的时候会直接创建cpu核数个p。一般是不会改变的
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
	lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
	return (*poolLocal)(lp)
}
```

```go 
// pin函数会拿到当前这个goroutineP对应的对象池
func (p *Pool) pin() *poolLocal {
	// 这个函数会拿到pid，
	// 同时会对m和p都加上锁
	// 防止因为goroutine切换导致当前go对应的p发生切换
	pid := runtime_procPin()
	
	// 这个地方是原子的获取p对象池的数量
	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	
	// 这个地址是不会改变的，所以没有进程原子操作
	l := p.local                          // load-consume
	
	// 这里直接就可以比较了
	// pid在生成的时候就是有序的
	// 那么在为每个p生成p对象池的时候就已经预留的小于当前pid对象池的空间了
	if uintptr(pid) < s { 
		return indexLocal(l, pid)
	}
	
	// In pinSlow we store to localSize and then to local, here we load in opposite order.
	// Since we've disabled preemption, GC cannot happen in between.
	// Thus here we must observe local at least as large localSize.
	// We can observe a newer/larger local, it is fine (we must observe its zero-initialized-ness).
	// 为什么是slow
	return p.pinSlow()
}
```

```go
// 初始化所有的p对象池
func (p *Pool) pinSlow() *poolLocal {
	// pinSlow入口只有pin，pin的时候已经获取了锁，这里解锁
	runtime_procUnpin()
	
	// 看名字就知道是所有的pool锁。
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	
	// 先解锁再加锁，避免同时持有两个锁
	// 1、避免死锁
	// 2、释放资源
	pid := runtime_procPin()
	
	// poolCleanup won't be called while we are pinned.
	// 这里没有原子操作？
	// 在pinSlow的时候不会进行poolCleanup，所以不需要原子操作？
	// 一会再看下poolCleanup
	s := p.localSize
	l := p.local
	
	// 入口只有pin，为什么这里又来了一遍
	// 1、上面已经执行了runtime_procPin，所以有可能这个p已经不是刚才的p了
	// 2、同时多个go在执行pinSlow，那么第一个执行完了，第二个就直接拿就好了
	if uintptr(pid) < s {
		return indexLocal(l, pid)
	}
	
	// 为什么是空？
	// 初始化，第一次使用，所以放入了全局的pool了
	if p.local == nil {
		allPools = append(allPools, p)
	}
	
	
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
	// 初始化，这个size就是p的数量了
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	
	// 原子存，非原子取
	// 存了就不会变了？？
	// 上面加锁了，不是原子的行不行？这个时候可能是有其他的go在读
	atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
	
	// 这里直接返回一个刚初始化，空的p对象池
	return &local[pid]
}
```
##### 到这里，已经知道了，在第一次put的时候，对象池才开始初始化。这次初始化会初始化p个p对象池
##### 每个p对象池，都有私有和共享的两种。
##### 为什么是私有和共享两种？
##### GET时，对于当前正在运行的p，如果曾经put了，直接获取就好了
##### 如果未put，这个共享是否就要使用了？

```go
func (p *Pool) Get() interface{} {
	// 这些流程都和put时是一样的
	l := p.pin()
	x := l.private
	l.private = nil
	runtime_procUnpin()
	
	// p本地已经没有私有的了
	if x == nil {
	
		// 从当前的p共享队列中获取
		l.Lock()
		last := len(l.shared) - 1
		if last >= 0 {
			x = l.shared[last]
			l.shared = l.shared[:last]
		}
		l.Unlock()
		
		// 获取不到就要getSLow
		if x == nil {
			x = p.getSlow()
		}
	}
	
	// 如果想尽一切办法都不能获取，就创建一个新的
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}

```

```go
func (p *Pool) getSlow() (x interface{}) {
	// 这个代码已经看见很多次了
	// 不同的是有的时候p对象池数量是原子获取，有的时候不是
	// 在pin的时候是原子的
	// 在pinSlow的时候不是原子的
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	local := p.local                         // load-consume
	
	// 获取当前的pid
	pid := runtime_procPin()
	runtime_procUnpin()
	
	// 依次遍历每一个p
	for i := 0; i < int(size); i++ {
	
		// 这个比较巧妙了
		// 因为runtime_procPin比较耗时，曲线救国
		l := indexLocal(local, (pid+i+1)%int(size))
		
		// 加锁，获取，解锁
		l.Lock()
		last := len(l.shared) - 1
		if last >= 0 {
			x = l.shared[last]
			l.shared = l.shared[:last]
			l.Unlock()
			break
		}
		l.Unlock()
	}
	return x
}
```

##### 到此，还有一个问题：不同的是有的时候p对象池数量是原子获取，有的时候不是

```go
// gc的时候，清理所有对象池
// 在stw的时候，gc开始的时候执行，
// 将所有的私有和共享的对象都至为nil

// 官方注释的解释就是防止错误保留整个对象池
// gc和put或者get同时发生的时候
// 未指明对象池的大小，在gc的时候只能把整个对象池清空了，防止对象池无限增长
// 这个在实际使用中应该是会有性能问题的。看别人在1.13有优化。

// 在stw的时候，不会有get或者put发生？
// 所以p对象池的数量永远都是非原子获取的
func poolCleanup() {
	// This function is called with the world stopped, at the beginning of a garbage collection.
	// It must not allocate and probably should not call any runtime functions.
	// Defensively zero out everything, 2 reasons:
	// 1. To prevent false retention of whole Pools.
	// 2. If GC happens while a goroutine works with l.shared in Put/Get,
	//    it will retain whole Pool. So next cycle memory consumption would be doubled.
	for i, p := range allPools {
		allPools[i] = nil
		for i := 0; i < int(p.localSize); i++ {
			l := indexLocal(p.local, i)
			l.private = nil
			for j := range l.shared {
				l.shared[j] = nil
			}
			l.shared = nil
		}
		p.local = nil
		p.localSize = 0
	}
	allPools = []*Pool{}
}
```

##### 对于最后一个问题：不同的是有的时候p对象池数量是原子获取，有的时候不是
什么时候会修改片对象池数量：pinSlow和poolCleanup，在执行pinSlow的时候，会所有对象池加一个大锁，所以保证只有一个pinSlow会执行。而在runtime_procPin会把p设置为不可抢占，这个时候就不会发生gc，pinSlow不是原子取p对象池数量也没有问题了。

**对象池是在用的时候才进行初始化的，也就是说，在没有执行put或者get时，是没有池子的概念的，这个和连接池都是类似的**

**通过share来进行不同的p中对象的流转的**

**因为存在私有的对象，所以还是会出现对象池中有对象，但是还要new的情况**

todo：

- runtime_procPin
- 1.13
