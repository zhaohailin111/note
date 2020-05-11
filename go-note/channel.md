# channel
 
**1.14.2**


```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// 数据类型大小相限制 1024*64 = 64k
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	
	// 类型检查 todo
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	// 计算需要分配的内存
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	// channel 是无缓冲区的，所以只需要分配chan机构体的内存
	case mem == 0: 		
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	
	// 因为元素不包含指针，可以直接计算元素大小，一次性分配
	case elem.ptrdata == 0:
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	
	
	default:
		c = new(hchan)
		// todo
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)	// 元素大小
	c.elemtype = elem					// 元素类型
	c.dataqsiz = uint(size)			// channel大小

	return c
}
```

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	
	// 非block操作，channel没有关闭。
	// 1、没有数据，并且当前channel没有等待接受数据的gr
	// 2、有数据，并且数据已写满
	// 发送失败
	// ch <- c是block，暂时不考虑，先走完发送流程
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	// 新特性？ 获取cpu的时间。
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	// 向一个关闭的channel发送数据
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
	
	// 从等待者队列拿出一个gr，如果拿得到，不经过数据缓冲区
	// sg是个双向链表。当gr阻塞，会打包成一个sudog结构
	if sg := c.recvq.dequeue(); sg != nil {
		// 唤醒这个gr
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 当前还有空间，非缓冲区的channel不会走这个逻辑
	// 有缓冲区的时候，当前的gr也不会阻塞。
	if c.qcount < c.dataqsiz {
		// 根据sender数量拿到数据写入的位置
		qp := chanbuf(c, c.sendx)
		
		// 写入数据
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		// todo
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	// todo
	if !block {
		unlock(&c.lock)
		return false
	}

	// 非缓冲区，或者缓冲区写满
	gp := getg()
	
	// sudog是否缓存的，并不是每次都需要新建一个sudug
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	
	// 设置一些参数
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	
	// 这个参数用于select
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	
	// 打包好，这时候就结束等待唤醒了
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// 唤醒以后要做的
	// 这个？
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if gp.param == nil {
		// 再次检查
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	
	// todo
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	
	// 回收sudog
	releaseSudog(mysg)
	return true
}

```
