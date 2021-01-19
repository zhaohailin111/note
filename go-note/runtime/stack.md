# 栈管理

## golang的栈也是在堆区


#### 为什么使用连续栈

##### 目前的分段栈存在以下问题：

- hot split：如果栈即将用完，调用函数的时候将会分配一块新的栈，函数返回以后这快栈就会被释放。循环调用开销大。
- 栈的扩容/缩容在分段栈上永远不会完成。超过阈值都要触发额外工作（这点不是很理解）

连续栈通过翻倍扩容解决这两个问题。防止频繁的创建/销毁

##### 连续栈的问题：

连续栈在扩容的时候为了保证连续，需要开辟一段新的连续的空间，然后将旧栈中数据移动到新的栈内。这样就存在两个情况：
- 堆中指针a1指向栈中数据A。转移以后A的地址变成了a2，那a1指向的就是旧的地址了，如果修改a1就要遍历所有的堆区指针，这显然不可能。
- 栈中指针指向栈地址，栈区移动的时候指向的地址也顺便移动，这个成本旧相对较低了。所以最大的问题就是堆指针指向栈地址。

golang通过约束：'栈地址只能被栈指针指向'来解决这个问题。所以：栈地址可以指向栈地址和堆地址，堆地址一定指向堆地址。不存在堆地址指向栈地址的情况。

##### 数据拷贝：

- 指数增长栈大小，分配一个新的栈区
- 旧栈区数据转移到新的栈区，旧栈区指针指向新的栈区。直接判断旧栈区的内容是不是旧栈区的边界。
- 整数类型用Carl判断（todo）
- 还有一些外部指针（函数调用相关）


```
// 栈结构，低地址高地址
type stack struct {
	lo uintptr
	hi uintptr
}

// 分配n子节地址的栈
// 栈复制/生成新的gr会调用
func stackalloc(n uint32) stack {
	// 分配栈区必须是在调度器的栈上
	// 判断是否是g0
	// 运行时代码经常会临时通过 systemstack, mcall 或 asmcgocall 切换到系统栈以执行那些无法扩展用户栈、或切换用户 goroutine 的不可抢占式任务。运行在系统栈上的代码是隐式不可抢占的、且不会被垃圾回收检测。当运行在系统栈上时，当前用户栈没有被用于执行代码。
	thisg := getg()
	if thisg != thisg.m.g0 {
		throw("stackalloc not on scheduler stack")
	}
	
	// n是否是2的倍数，新创建的g是2048
	if n&(n-1) != 0 {
		throw("stack size not a power of 2")
	}

	// Small stacks are allocated with a fixed-size free-list allocator.
	// If we need a stack of a bigger size, we fall back on allocating
	// a dedicated span.
	
	// 小栈区的分配和大栈区分配
	var v unsafe.Pointer
	
	// 不通系统的小栈区不一样，第二个是32k
	if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
	
		// 增长的次数
		order := uint8(0)
		n2 := n
		for n2 > _FixedStack {
			order++
			n2 >>= 1
		}
		var x gclinkptr
		
		// m的缓存
		// 内存分配都会为每个m分配缓存
		c := thisg.m.mcache
		
		// 没有缓存的情况会去全局pool中分配
		if stackNoCache != 0 || c == nil || thisg.m.preemptoff != "" {
			// c == nil can happen in the guts of exitsyscall or
			// procresize. Just get a stack from the global pool.
			// Also don't touch stackcache during gc
			// as it's flushed concurrently.
			
			// 全局缓存是有锁的
			lock(&stackpool[order].item.mu)
			x = stackpoolalloc(order)
			unlock(&stackpool[order].item.mu)
			
			// 如果线程本地是有缓存的，就直接在本地拿到对应的空间
			// stackcache会对每个order生成一个列表
		} else {
			// 线程缓存是无锁的
			x = c.stackcache[order].list
			if x.ptr() == nil {
			
				// 如果本地不足，补充缓存
				stackcacherefill(c, order)
				x = c.stackcache[order].list
			}
			// 从缓存链表头节点获取一个
			c.stackcache[order].list = x.ptr().next
			c.stackcache[order].size -= uintptr(n)
		}
		v = unsafe.Pointer(x)
		
		// 超过32k的直接去大栈区分配
	} else {
		var s *mspan
		
		// 根据n的大小获取对应的页。
		npage := uintptr(n) >> _PageShift
		log2npage := stacklog2(npage)

		// Try to get a stack from the large stack cache.
		// 全局栈缓存获取
		lock(&stackLarge.lock)
		// 不为空直接获取
		if !stackLarge.free[log2npage].isEmpty() {
			s = stackLarge.free[log2npage].first
			stackLarge.free[log2npage].remove(s)
		}
		unlock(&stackLarge.lock)
	
		// 为空分配一个新的
		if s == nil {
			// Allocate a new stack from the heap.
			// 从堆中分配一个span
			s = mheap_.allocManual(npage, &memstats.stacks_inuse)
			if s == nil {
				throw("out of memory")
			}
			osStackAlloc(s)
			s.elemsize = uintptr(n)
		}
		v = unsafe.Pointer(s.base())
	}

	return stack{uintptr(v), uintptr(v) + uintptr(n)}
}

// 从全局缓存里面分配一部分到线程缓存
// 每次只分配一半，这样归还的时候也不会立刻溢出吧
func stackcacherefill(c *mcache, order uint8) {
	// Grab some stacks from the global cache.
	// Grab half of the allowed capacity (to prevent thrashing).
	var list gclinkptr
	var size uintptr
	lock(&stackpool[order].item.mu)
	for size < _StackCacheSize/2 {
		x := stackpoolalloc(order)
		x.ptr().next = list
		list = x
		size += _FixedStack << order
	}
	unlock(&stackpool[order].item.mu)
	c.stackcache[order].list = list
	c.stackcache[order].size = size
}

// 从全局缓存拿一个_FixedStack << order大小的内存
func stackpoolalloc(order uint8) gclinkptr {
	// 从全局缓存拿一个第一个span
	list := &stackpool[order].item.span
	s := list.first
	
	// 没有空闲的span
	if s == nil {
		// no free stacks. Allocate another span worth.
		// 从堆中分配一个span
		s = mheap_.allocManual(_StackCacheSize>>_PageShift, &memstats.stacks_inuse)
		if s == nil {
			throw("out of memory")
		}
		if s.allocCount != 0 {
			throw("bad allocCount")
		}
		if s.manualFreeList.ptr() != nil {
			throw("bad manualFreeList")
		}
		osStackAlloc(s)
		
		// 将span分割成若干个_FixedStack << order大小的块依次串起来
		s.elemsize = _FixedStack << order
		for i := uintptr(0); i < _StackCacheSize; i += s.elemsize {
			x := gclinkptr(s.base() + i)
			x.ptr().next = s.manualFreeList
			s.manualFreeList = x
		}
		
		// 插入到span的列表
		list.insert(s)
	}
	
	// 拿到第一个然后返回
	x := s.manualFreeList
	if x.ptr() == nil {
		throw("span has no free stacks")
	}
	s.manualFreeList = x.ptr().next
	s.allocCount++
	if s.manualFreeList.ptr() == nil {
		// all stacks in s are allocated.
		list.remove(s)
	}
	return x
}
```

##### 可以看到两个缓存：线程缓存，全局缓存。
##### 线程缓存只分配小的栈。全局缓存分为小栈和大栈
##### 当栈的大小小于阈值，依次从线程缓存->全局缓存里面拿。如果线程缓存拿不到，从全局转移一半到线程缓存。全局缓存如果也没有，从堆中分配一个span给到全局缓存。线程是无锁的，全局是有锁的。
##### 当栈的大小大于阈值，直接去全局缓存里面拿

```
// 通过这个来检查是否需要扩容
func newstack() {
	// 当前执行调度的g
	thisg := getg()

	// 即将执行的g
	gp := thisg.m.curg

	// 判断是否分段栈，这个应该是不合法的吧
	if thisg.m.curg.throwsplit {
		// Update syscallsp, syscallpc in case traceback uses them.
		morebuf := thisg.m.morebuf
		gp.syscallsp = morebuf.sp
		gp.syscallpc = morebuf.pc
		pcname, pcoff := "(unknown)", uintptr(0)
		f := findfunc(gp.sched.pc)
		if f.valid() {
			pcname = funcname(f)
			pcoff = gp.sched.pc - f.entry
		}
		print("runtime: newstack at ", pcname, "+", hex(pcoff),
			" sp=", hex(gp.sched.sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
			"\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
			"\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")

		thisg.m.traceback = 2 // Include runtime frames
		traceback(morebuf.pc, morebuf.sp, morebuf.lr, gp)
		throw("runtime: stack split at bad time")
	}

	// 初始化寄存器信息？
	morebuf := thisg.m.morebuf
	thisg.m.morebuf.pc = 0
	thisg.m.morebuf.lr = 0
	thisg.m.morebuf.sp = 0
	thisg.m.morebuf.g = 0
	
	// 当前的g是否发起了抢占
	preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

	// Be conservative about where we preempt.
	// We are interested in preempting user Go code, not runtime code.
	// If we're holding locks, mallocing, or preemption is disabled, don't
	// preempt.
	// This check is very early in newstack so that even the status change
	// from Grunning to Gwaiting and back doesn't happen in this case.
	// That status change by itself can be viewed as a small preemption,
	// because the GC might change Gwaiting to Gscanwaiting, and then
	// this goroutine has to wait for the GC to finish before continuing.
	// If the GC is in some way dependent on this goroutine (for example,
	// it needs a lock held by the goroutine), that small preemption turns
	// into a real deadlock.
	
	// 如果发起抢占
	if preempt {
		if !canPreemptM(thisg.m) {
			// Let the goroutine keep running for now.
			// gp->preempt is set, so it will be preempted next time.
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	if gp.stack.lo == 0 {
		throw("missing stack in newstack")
	}
	sp := gp.sched.sp
	if sys.ArchFamily == sys.AMD64 || sys.ArchFamily == sys.I386 || sys.ArchFamily == sys.WASM {
		// The call to morestack cost a word.
		sp -= sys.PtrSize
	}
	if stackDebug >= 1 || sp < gp.stack.lo {
		print("runtime: newstack sp=", hex(sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
			"\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
			"\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")
	}
	if sp < gp.stack.lo {
		print("runtime: gp=", gp, ", goid=", gp.goid, ", gp->status=", hex(readgstatus(gp)), "\n ")
		print("runtime: split stack overflow: ", hex(sp), " < ", hex(gp.stack.lo), "\n")
		throw("runtime: split stack overflow")
	}

	if preempt {
		if gp == thisg.m.g0 {
			throw("runtime: preempt g0")
		}
		if thisg.m.p == 0 && thisg.m.locks == 0 {
			throw("runtime: g is running but p is not")
		}

		if gp.preemptShrink {
			// We're at a synchronous safe point now, so
			// do the pending stack shrink.
			gp.preemptShrink = false
			shrinkstack(gp)
		}

		if gp.preemptStop {
			preemptPark(gp) // never returns
		}

		// Act like goroutine called runtime.Gosched.
		gopreempt_m(gp) // never return
	}

	// Allocate a bigger segment and move the stack.
	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize * 2
	if newsize > maxstacksize {
		print("runtime: goroutine stack exceeds ", maxstacksize, "-byte limit\n")
		print("runtime: sp=", hex(sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n")
		throw("stack overflow")
	}

	// The goroutine must be executing in order to call newstack,
	// so it must be Grunning (or Gscanrunning).
	casgstatus(gp, _Grunning, _Gcopystack)

	// The concurrent GC will not scan the stack while we are doing the copy since
	// the gp is in a Gcopystack status.
	copystack(gp, newsize)
	if stackDebug >= 1 {
		print("stack grow done\n")
	}
	casgstatus(gp, _Gcopystack, _Grunning)
	gogo(&gp.sched)
}
```


[https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)

[https://changkun.us/archives/2018/09/255/](https://changkun.us/archives/2018/09/255/)

[https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/)
