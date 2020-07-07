# sync.WaitGroup


```go
type WaitGroup struct {
	// wg只能传地址拷贝，不能传值
	noCopy noCopy

	// 共12字节
	// 其中8字节进行原子操作，4字节表示正在执行的gr数量，也就是调用Add函数的gr数量。另外4字节表示等待者的数量，也就是调用Wait函数的gr数量。
	// 另外4字节表示等待者队列。
	state1 [3]uint32
}
```

```go
// 因为字节对齐的原因，如果结构体起始位置%8==0，说明是64位
// 分别返回的是进行原子操作的64位和等待着队列的32位
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}

func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	
	// delta是偏移量,可能是负数。
	// 当delta是-1的时候，转成uint64为11111111,+1以后变成0，+2以后变成1
	// state是加完的结果
	// 当delta为正数的时候，相当于state += (1 << 32)
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	// wait等的那些gr的数量
	v := int32(state >> 32)
	// 调用wait的gr数量
	w := uint32(state)

	// v>0表示，还有gr没做完，调用wait的gr还要接着等
	// w==0表示，gr都做完了，同时调用wait的gr数量为0，wg恢复到初始化的状态了。什么都不用做了
	if v > 0 || w == 0 {
		return
	}
	// 这里表示v == 0 && w > 0 
	// 也就是说，所有调用Add的gr都已经Done了，但是调用Wait的gr还在等着呢
	*statep = 0
	for ; w != 0; w-- {
		// 依次唤醒调用Wait的gr
		runtime_Semrelease(semap, false, 0)
	}
}

// Done就相当于state -= (1 << 32)
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}

func (wg *WaitGroup) Wait() {
	// 获取地址
	statep, semap := wg.state()

	for {
		// 拿到状态值
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		w := uint32(state)
		// 如果v等于0的话，说明没有gr调用Add或者所有调用Add的gr都已经调用Done了，不需要等待了。
		if v == 0 {
			return
		}
		// Increment waiters count.
		// 自增等待者的数量
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			// 自增成功，休眠
			runtime_Semacquire(semap)
			return
		}
	}
}
```

