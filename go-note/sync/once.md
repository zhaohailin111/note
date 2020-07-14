# sync.Once

多次调用Once.Do只有第一次才会执行，无论多次调用的方法是否相同

```
// 结构包含一个进行原子操作的uint32和一个互斥锁
type Once struct {
	done uint32
	m    Mutex
}

// do只接收一个func
// 第一次看到这个once包以为只是进行一次cas操作判断是否为真就可以了
// 实际看到的竟然不是这样简单的
// 作者直接注释给出了有一种错误的实现以及其问题
// 当两个gr同时调用Do执行一个set操作
// 第一个gr正在执行中
// 第二个gr cas失败进行后续的一个get操作
// 就出现了异常，明明set已经执行了，但是get还没有拿到最新的扼数据
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.
	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}

// 逻辑也很简单，加锁保证第二个gr一定是在第一个gr结束以后才结束。
func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}

```

##### 参考文献：
[https://colobu.com/2020/07/05/the-state-of-go-sync-2020/](ttps://colobu.com/2020/07/05/the-state-of-go-sync-2020/)
