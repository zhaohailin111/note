# channel
 
**1.14.2**


```go
type hchan struct {
	qcount   uint           // channel中数据总数
	dataqsiz uint           // buf的大小
	buf      unsafe.Pointer // 存储channel中的数据
	elemsize uint16
	closed   uint32 // 是否关闭
	elemtype *_type // 存储的元素类型
	sendx    uint   // 等待发送的gr数量
	recvx    uint   // 等待接收的gr数量
	// 接收/发送的gr列表。每一个都是个双向链表
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters
	lock mutex
}

// 存储gr的双向链表
type waitq struct {
	first *sudog // gr阻塞都会被打包成一个sudog存储起来
	last  *sudog
}

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
	// channel 是无缓冲区的，所以只需要分配chan结构体的内存
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
// entry point for c <- x from compiled code
// c<-x的入口
func chansend1(c *hchan, elem unsafe.Pointer) {
	// 第三个参数是是否阻塞，在写入的时候如果channel缓存区不足，就会阻塞
	chansend(c, elem, true, getcallerpc())
}

// 第三个参数表示，此次操作是否会block
// 正常向chanel写入数据的时候，缓冲区写满是会阻塞的
// select时即使写不进去也不会立刻阻塞
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	
	// 一个未初始化的chan
	if c == nil {
		// 正常向chan中发送数据的时候都是阻塞的
		// 在select时向channel发送是非阻塞的
		// 因为chan没有初始化，所以返回写入失败
		if !block {
			return false
		}
		// 否则阻塞当前这个gr
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
	
	// 非block操作，channel没有关闭。
	// 1、无缓冲区的chan，并且当前channel没有等待接受数据的gr	// 2、有缓冲区但是chan已满，
	// 这时候chan写不进数据，又不能阻塞，返回false
	// 用来加速select。以上两种情况都是不会被select的
	if !block && c.closed == 0 && 
	((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	// 新特性？ 获取cpu的时间。
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 接下来是有锁操作
	lock(&c.lock)

	// 向一个关闭的channel发送数据，panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
	
	// 从等待者队列拿出一个gr，如果拿得到，直接将数据转移给他，并且带着锁
	// chan就像是一个箱子。一边等着放，一边等着拿
	// 如果放的人发现正在有人等着拿，说明箱子是空的，直接就交付给对方
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

	// 到这里不管是否是有缓冲区，都是写入失败了。
	// 如果不能阻塞，返回失败
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
	// 因为select是不阻塞的，所以一定不会走到这里了
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	// 这个param是会在被唤醒的时候设置为这个g对应打包的sudug
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
	// 唤醒后发现本来打包我的sudog不是那个sudog了
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	// 出了close，被唤醒的时候这个param应该是打包这个g的sudog
	// 如果是正常接收者去唤醒一个发送者，那这个param是对应的发送者，select会用到
	if gp.param == nil {
		// 再次检查
		// 异常唤醒
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		// 唤醒的时候可能是已经关闭了，这个时候会panic
		// 缓冲区已满的情况发送是会阻塞
		// 此时chan上有若干个发送者，关闭chan会走到这里触发panic
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

// 发送时chan的接收者队列能拿到gr
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if raceenabled {
		if c.dataqsiz == 0 {
			racesync(c, sg)
		} else {
			// Pretend we go through the buffer, even though
			// we copy directly. Note that we need to increment
			// the head/tail locations only when raceenabled.
			qp := chanbuf(c, c.recvx)
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
			c.recvx++
			if c.recvx == c.dataqsiz {
				c.recvx = 0
			}
			c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
		}
	}
	// 如果有数据，转移数据
	// channel在发送空结构体这个应该是空的
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	// 解锁channel
	// 这时候
	// 但其实在send函数入口处就可以解锁了吧？
	unlockf()
	// 发送者也会用这个传递给接收者
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	// 唤醒这个g
	goready(gp, skip+1)
}

```

```go
// 关闭channel，把队列里面的gr唤醒就好了
func closechan(c *hchan) {
	// 关闭一个空的channel会panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	// 关闭一个已经被关闭的chan会panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

	// 设置为已关闭
	c.closed = 1

	// 用来保存所有接收者和发送者
	var glist gList

	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		// 关闭chan的时候把他设置为nil，唤醒的时候进行检查
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		// 因为发生panic的gr不是close这个chan的gr
		// 所以在这里是不会panic的，而是发送者被唤醒的时候panic
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	// 依次唤醒
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```


```go
// 从chan接收数据分为两种情况
// a <- c
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

// a, ok <- c
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	// select一个nil的chan是失败的
	if c == nil {
		if !block {
			return
		}
		// 允许block，则阻塞
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// 很多逻辑和发送端是一样的
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	// 已经关闭的channel，且每数据了
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	// 有发送者等着呢
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 有数据，直接拿数据
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	// 不能阻塞，返回失败
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 准备进入阻塞
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// 根据这个返回chan是否关闭了
	// 这样是无锁操作
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}

func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
		}
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	// 可以看到，因为接收而去唤醒gr，这里的param不是空的
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}

// 发送和接收大体流程都是一样的
```

```go
func reflect_rselect(cases []runtimeSelect) (int, bool) {
	// select的case是空的，直接阻塞forever
	if len(cases) == 0 {
		block()
	}
	sel := make([]scase, len(cases))
	order := make([]uint16, 2*len(cases))
	for i := range cases {
		rc := &cases[i]
		switch rc.dir {
		case selectDefault: // default
			sel[i] = scase{kind: caseDefault}
		case selectSend: // case <-Chan:
			sel[i] = scase{kind: caseSend, c: rc.ch, elem: rc.val}
		case selectRecv: // case Chan <- Send
			sel[i] = scase{kind: caseRecv, c: rc.ch, elem: rc.val}
		}
		if raceenabled || msanenabled {
			selectsetpc(&sel[i])
		}
	}

	return selectgo(&sel[0], &order[0], len(cases))
}

// select的代码一大坨
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	
	// 将传入的case列表强转
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	// select的case列表
	scases := cas1[:ncases:ncases]
	// select的顺序
	pollorder := order1[:ncases:ncases]
	// 给每个chan加锁时的顺序
	lockorder := order1[ncases:][:ncases:ncases]

	// 将那些空的chan给一个默认值
	for i := range scases {
		cas := &scases[i]
		// nil的chan
		if cas.c == nil && cas.kind != caseDefault {
			*cas = scase{}
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
		for i := 0; i < ncases; i++ {
			scases[i].releasetime = -1
		}
	}


	// 调换顺序，select随机的原因
	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[i] = pollorder[j]
		pollorder[j] = uint16(i)
	}

	// sort the cases by Hchan address to get the locking order.
	// simple heap sort, to guarantee n log n time and constant stack footprint.
	for i := 0; i < ncases; i++ {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	for i := ncases - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}
	
	// 锁定所有chan
	sellock(scases, lockorder)

	var (
		gp     *g
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

loop:
	// pass 1 - look for something already waiting
	var dfli int
	var dfl *scase
	var casi int
	var cas *scase
	var recvOK bool
	for i := 0; i < ncases; i++ {
		// 根据pollorder来保证乱序的
		casi = int(pollorder[i])
		cas = &scases[casi]
		c = cas.c

		switch cas.kind {
		// 空的chan，什么都不做
		case caseNil:
			continue
		
		// 从chan读
		case caseRecv:
			sg = c.sendq.dequeue()
			if sg != nil {
				// 发送者队列不是空的，跳到接收
				goto recv
			}
			if c.qcount > 0 {
				// 发送者队列时空的，但是有数据
				goto bufrecv
			}
			if c.closed != 0 {
				// channel被关闭了，可以获取默认值
				goto rclose
			}

		// 向chan发
		case caseSend:
			if raceenabled {
				racereadpc(c.raceaddr(), cas.pc, chansendpc)
			}
			if c.closed != 0 {
				// 向一个关闭的chan发送数据
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				// 有接收者等着，跳到发送
				goto send
			}
			if c.qcount < c.dataqsiz {
				// 缓冲区足够，可以发送
				goto bufsend
			}
		
		// 不会立刻结束循环，而只是做个记录
		// 还是要从其他的case中寻找
		case caseDefault:
			// select default
			dfli = casi
			dfl = cas
		}
	}

	// 没有case可以响应，走default
	if dfl != nil {
		selunlock(scases, lockorder)
		casi = dfli
		cas = dfl
		goto retc
	}

	// pass 2 - enqueue on all chans
	gp = getg()
	// 异常，有sudog在等待这个g？
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	// 这个就很牛逼了
	nextp = &gp.waiting
	// 此时没有case可以通过，并且没有default
	// 需要把当前的这个g挂在所有的chan下面
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		// 空的chan就不需要挂了
		// select 空的chan，这个case就永远不会成功了
		if cas.kind == caseNil {
			continue
		}
		// 拿到这个case对应的chan
		c = cas.c
		// 打包成sudog
		sg := acquireSudog()
		sg.g = gp
		// 当前这个sudog标记为因为select而阻塞，这个很重要
		sg.isSelect = true
		// No stack splits between assigning elem and enqueuing
		// sg on gp.waiting where copystack can find it.
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// Construct waiting list in lock order.
		// 这个地方串成一个列表。
		// 把上一个sudog的nextp指向当前这这个sudog
		*nextp = sg
		// 当前这个wailink就会在下一次循环指向下一个sudog
		// 这样唤醒g的时候就可以拿到gp.waiting
		// 而gp.waiting就是因select导致打包的sudog的链表的头节点
		nextp = &sg.waitlink

		// 根据case的类型依次挂在到对应chan上
		switch cas.kind {
		case caseRecv:
			c.recvq.enqueue(sg)

		case caseSend:
			c.sendq.enqueue(sg)
		}
	}

	// wait for someone to wake us up
	// 阻塞当前这个gr，等待唤醒
	gp.param = nil
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)
	
	// 被唤醒了
	gp.activeStackChans = false

	sellock(scases, lockorder)

	gp.selectDone = 0
	// 之前说gr唤醒时，这个param就是打包当前这个gr的sudog，也就是sg打包当前这个gr
	// 但是因为select其实是将当前的gr打包了很多sudog放在各个case的chan上面
	// 而之前说了gp.waiting刚好把刚才打包的很多个sudog串起来了
	sg = (*sudog)(gp.param)
	// 这里没有判断param是否是空，也就是没有判断是否是异常唤醒
	// 之前chan部分有看到，关闭chan导致唤醒这个地方就是nil了
	gp.param = nil

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
	casi = -1
	cas = nil
	
	// 这个地方就是刚才串起来的
	// 而上面的那个sg就是对应的唤醒的sudog
	// 也就是说，唤醒的时候，只知道是唤醒了哪个gr，但是并不知道是哪个sudog
	// 通过sglist和sg就可以找到到底是哪个case唤醒的了。简直太牛逼了
	// 太牛逼了，太牛逼了，太牛逼了！！！
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	// 根据gp.waiting得到的sudog链表。依次设置状态为false。
	// 唤醒了就可以把之前打包在各个chan的sudog拿走了
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	// 对于每个case，把各个chan的sudog删除掉
	for _, casei := range lockorder {
		k = &scases[casei]
		if k.kind == caseNil {
			continue
		}
		if sglist.releasetime > 0 {
			k.releasetime = sglist.releasetime
		}
		// 找到唤醒他的sudog
		// 因为sg可能是nil。所以这个地方可能不会走到
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			casi = int(casei)
			cas = k
		} else {
			// 如果不是唤醒他的sudog，从对应的chan队列sudog拿走
			c = k.c
			if k.kind == caseSend {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		// 依次获取sudog
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		// 释放这个sudog
		releaseSudog(sglist)
		sglist = sgnext
	}
	
	// cas为nil，说明param是nil，说明因为closechan导致的gr被唤醒
	// goto loop 就可以发现对应被关闭的chan了
	// 这里因为要走上面那一堆逻辑（把chan上的sudog拿走），所以不能直接判断param=nil就goto loop
	if cas == nil {
		// We can wake up with gp.param == nil (so cas == nil)
		// when a channel involved in the select has been closed.
		// It is easiest to loop and re-run the operation;
		// we'll see that it's now closed.
		// Maybe some day we can signal the close explicitly,
		// but we'd have to distinguish close-on-reader from close-on-writer.
		// It's easiest not to duplicate the code and just recheck above.
		// We know that something closed, and things never un-close,
		// so we won't block again.
		goto loop
	}

	// 找到对应唤醒他的sudog了
	c = cas.c

	// 如果这个case是从chan接收，第二个参数变成true
	if cas.kind == caseRecv {
		recvOK = true
	}

	// 解锁
	selunlock(scases, lockorder)
	goto retc

// 从buffer获取
bufrecv:
	// can receive from buffer
	if raceenabled {
		if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, cas.pc, chanrecvpc)
		}
		raceacquire(chanbuf(c, c.recvx))
		racerelease(chanbuf(c, c.recvx))
	}
	if msanenabled && cas.elem != nil {
		msanwrite(cas.elem, c.elemtype.size)
	}
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc

// 向buffer发送
bufsend:
	// can send to buffer
	if raceenabled {
		raceacquire(chanbuf(c, c.sendx))
		racerelease(chanbuf(c, c.sendx))
		raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.size)
	}
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc

// 直接接收，需要唤醒某个发送者
recv:
	// can receive from sleeping sender (sg)
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	recvOK = true
	goto retc

// 从已经关闭的chan读取数据
rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	if raceenabled {
		raceacquire(c.raceaddr())
	}
	goto retc

// 直接发送，需要唤醒某个接收者
send:
	// can send to a sleeping receiver (sg)
	if raceenabled {
		raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.size)
	}
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc

// 返回数据
retc:
	if cas.releasetime > 0 {
		blockevent(cas.releasetime-t0, 1)
	}
	return casi, recvOK

// 向关闭的chan发送数据
sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```
