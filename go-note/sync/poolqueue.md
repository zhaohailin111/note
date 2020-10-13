# sync.poolqueue
- 给sync.pool使用的无锁队列

- 队列支持并发消费，但是不支持并发写入。 
- 这一点是对象池的特性导致，在当前这个g对应的p上，不会出现并发写入和读取的，一个p只能绑定一个g。但是在共享队列上，是支持并发读的，当前p上没有了，需要上其他的p上拿，所以会出现多个g去同一个p上拿的情况。
- 所以，这个无锁队列提供了三个操作，pushHead，popHead，不支持并发操作。popTail，支持并发操作。
- pushHead和popHead是给当前g对应的这个p上的队列操作的，popTail是给其他的g在当前这个队列上读取的。

- 整个pool使用的队列包括两种结构：定长和变长队列，变长队列是在定长队列的基础上实现的
- 定长队列在创建时就不会扩容。变长队列会将若干个定长队列串成链表组成变长队列，变长队列每次扩容都是扩两倍的上次申请的定长队列长度。

#### 定长队列
```go
type poolDequeue struct {
	
	// 队列的头尾指针，高32位是队头，低32位是队位
	// head表示队列中下一个节点的位置
	// tail表示队列中最旧的节点的位置
	// 使用的时候head和tail都会对队列长度取模
	headTail uint64
	
	// 环形链表
	// 链表长度都是预先开辟好的，为2的倍数，这个和切片都是一样的
	// 链表2的倍数取模只要&长度-1就可以了
	vals []eface
}

// 环形链表中的每个节点
type eface struct {
	// 这里的typ有什么用呢？
	typ, val unsafe.Pointer
}

const dequeueBits = 32

// 队列最大长度
// 32位最大就是2^32-1，这里队列最大长度是2^30次方，因为head和tail在队列满的时候不会变小，所以这里最大不是2^31
const dequeueLimit = (1 << dequeueBits) / 4

// 队列空节点
type dequeueNil *struct{}

``` 

```go
// 将headTail转为head和tail
func (d *poolDequeue) unpack(ptrs uint64) (head, tail uint32) {
	const mask = 1<<dequeueBits - 1 // 32个1
	head = uint32((ptrs >> dequeueBits) & mask) // 高32位
	tail = uint32(ptrs & mask) // 低32位
	return
}

// 将head和tail转为headTail
func (d *poolDequeue) pack(head, tail uint32) uint64 {
	const mask = 1<<dequeueBits - 1
	return (uint64(head) << dequeueBits) |
		uint64(tail&mask)
}
```

```go
// 向队头插入节点
// 只有队列生产者可以调用
func (d *poolDequeue) pushHead(val interface{}) bool {
	ptrs := atomic.LoadUint64(&d.headTail)
	// 获取头尾指针
	head, tail := d.unpack(ptrs)
	
	// 判断队列是否是满的，这里队列满的。tail+队列长度如果等于head，就说明队列满了。
	// len(vals)并不是队列中实际元素个数，而是空间大小。队列会预开辟一部分空间
	// eg：队头推入十个，队位拿出6个，head=10，tail=6，len(vals)=8，这时候没满，再推4个，这时候head=14就是满了。
	if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
		// Queue is full.
		return false
	}
	
	// head%len(vals)
	slot := &d.vals[head&uint32(len(d.vals)-1)]

	// Check if the head slot has been released by popTail.
	// 这个地方是个极端情况，后面整体看
	typ := atomic.LoadPointer(&slot.typ)
	if typ != nil {
		// Another goroutine is still cleaning up the tail, so
		// the queue is actually still full.
		return false
	}

	// The head slot is free, so we own it.
	if val == nil {
		val = dequeueNil(nil)
	}
	*(*interface{})(unsafe.Pointer(slot)) = val

	// 因为head是高32位，所以这里加2^32，相当于加1
	atomic.AddUint64(&d.headTail, 1<<dequeueBits)
	return true
}

// 只有队列生产者可以调用
func (d *poolDequeue) popHead() (interface{}, bool) {
	var slot *eface
	// 循环尝试出队
	for {
		ptrs := atomic.LoadUint64(&d.headTail)
		head, tail := d.unpack(ptrs)
		// 头尾指针相同，队列是空的
		if tail == head {
			// Queue is empty.
			return nil, false
		}

		// Confirm tail and decrement head. We do this before
		// reading the value to take back ownership of this
		// slot.
		
		// 这个地方不能使用原子减，必须用cas
		// 但是入队可以，入队是对头前移，不会冲突
		// 但是出队是对头后移，而此时可能存在队尾前移，导致获取到同一个节点，所以使用cas
		head--
		ptrs2 := d.pack(head, tail)
		if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
			// We successfully took back slot.
			slot = &d.vals[head&uint32(len(d.vals)-1)]
			break
		}
	}
	
	// 获取val
	val := *(*interface{})(unsafe.Pointer(slot))
	if val == dequeueNil(nil) {
		val = nil
	}
	// Zero the slot. Unlike popTail, this isn't racing with
	// pushHead, so we don't need to be careful here.
	
	// 节点重置为空
	*slot = eface{}
	// 到这里是否还有并发问题，popHead和popTail同时的问题，cas可以解决
	// popHead和pushHead不会并发使用，所以没有并发问题了
	return val, true
}

// popTail removes and returns the element at the tail of the queue.
// It returns false if the queue is empty. It may be called by any
// number of consumers.
func (d *poolDequeue) popTail() (interface{}, bool) {
	var slot *eface
	// 这个流程和popHead一样的
	for {
		ptrs := atomic.LoadUint64(&d.headTail)
		head, tail := d.unpack(ptrs)
		if tail == head {
			// Queue is empty.
			return nil, false
		}

		// Confirm head and tail (for our speculative check
		// above) and increment tail. If this succeeds, then
		// we own the slot at tail.
		ptrs2 := d.pack(head, tail+1)
		if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
			// Success.
			slot = &d.vals[tail&uint32(len(d.vals)-1)]
			break
		}
	}

	// We now own slot.
	// 获取val
	val := *(*interface{})(unsafe.Pointer(slot))
	if val == dequeueNil(nil) {
		val = nil
	}

	// Tell pushHead that we're done with this slot. Zeroing the
	// slot is also important so we don't leave behind references
	// that could keep this object live longer than necessary.
	//
	// We write to val first and then publish that we're done with
	// this slot by atomically writing to typ.
	// 这里是区别，popTail的时候需要原子的存储typ而不是*slot = eface{}
	slot.val = nil
	atomic.StorePointer(&slot.typ, nil)
	// At this point pushHead owns the slot.

	return val, true
}
```
##### 上面还有一个问题，节点中的typ有什么用？
- 首先看popHead和popTail的区别，popHead只需要*slot = eface{}，但是popTail是先slot.val = nil，在修改typ，这里popHead很好解释，不会出现同时操作当前节点的情况，因为即使出现并行操作，也拿不到同一个节点，但是pushheah和popTail是存在操作同一个节点的情况的，因为pushHead没有使用cas
	- 先pushHead在popTail导致获取同一个节点可能吗？假如长度是8，head=1，tail=0，这时候pushHead拿到的是1，但是popTail是不可能拿到1的，因为tail=head就返回空，tail又不能超过head
	- 先popTail在pushHead呢？咋一看也是不能的，原因也是一样的。但是队列是环形的，也就是存在len=8，tail=0，head=8，此时同时出队8个，但是tail=0的那个节点还没处理呢，head=8的那个节点已经写入数据了。这时候就出现操作同一个节点的情况。在pushHead上的判断```if typ != nil {```在这个场景出现的表现就是队列已经满了。这时候要先保证在popTail的时候把val先处理掉，在去修改typ。

#### 变长队列
```go

// 变长队列由两个chain组成
type poolChain struct {
	head *poolChainElt

	tail *poolChainElt
}


// 每个chain都是一个定长队列加上前后指针，组成双向链表
type poolChainElt struct {
	poolDequeue

	// next and prev link to the adjacent poolChainElts in this
	// poolChain.
	//
	// next is written atomically by the producer and read
	// atomically by the consumer. It only transitions from nil to
	// non-nil.
	//
	// prev is written atomically by the consumer and read
	// atomically by the producer. It only transitions from
	// non-nil to nil.
	next, prev *poolChainElt
}

// 原子存储chain
func storePoolChainElt(pp **poolChainElt, v *poolChainElt) {
	atomic.StorePointer((*unsafe.Pointer)(unsafe.Pointer(pp)), unsafe.Pointer(v))
}

// 原子获取chain
func loadPoolChainElt(pp **poolChainElt) *poolChainElt {
	return (*poolChainElt)(atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(pp))))
}

// 这个是不会并发调用的
func (c *poolChain) pushHead(val interface{}) {
	// 初始化队头
	d := c.head
	if d == nil {
		// 初始化队列节点
		// 初始长度是8，每扩容一次，都是*2扩容
		const initSize = 8 // Must be a power of 2
		d = new(poolChainElt)
		d.vals = make([]eface, initSize)
		c.head = d
		storePoolChainElt(&c.tail, d)
	}
	
	// 向定长队列添加节点
	if d.pushHead(val) {
		return
	}
	
	// 添加失败，上一个定长队列已经满了，生成一个新的节点，两倍容量
	newSize := len(d.vals) * 2
	if newSize >= dequeueLimit {
		// 不能超过最大值
		newSize = dequeueLimit
	}

	d2 := &poolChainElt{prev: d}
	d2.vals = make([]eface, newSize)
	c.head = d2
	storePoolChainElt(&d.next, d2)
	d2.pushHead(val)
}

func (c *poolChain) popHead() (interface{}, bool) {
	d := c.head
	for d != nil {
		if val, ok := d.popHead(); ok {
			return val, ok
		}
		// There may still be unconsumed elements in the
		// previous dequeue, so try backing up.
		d = loadPoolChainElt(&d.prev)
	}
	return nil, false
}

func (c *poolChain) popTail() (interface{}, bool) {
	// tail是空的，说明队列是空的
	d := loadPoolChainElt(&c.tail)
	if d == nil {
		return nil, false
	}

	for {
		// It's important that we load the next pointer
		// *before* popping the tail. In general, d may be
		// transiently empty, but if next is non-nil before
		// the pop and the pop fails, then d is permanently
		// empty, which is the only condition under which it's
		// safe to drop d from the chain.
		// 下一个定长队列
		// 这里d2一定是要在pop之前获取，为什么呢？
		d2 := loadPoolChainElt(&d.next)
		// 如果当前这个定长队列有元素可以pop，直接返回
		if val, ok := d.popTail(); ok {
			return val, ok
		}
		// 如果没有，看下一个定长队列是否为nil
		// 至少可以确定，在这一时刻，整个队列是空的，同时这个队列只有一个为空的定长数组，说明队列处于刚刚初始化的状态
		if d2 == nil {
			// This is the only dequeue. It's empty right
			// now, but could be pushed to in the future.
			return nil, false
		}

		// 到这里，可以确定tail这个空的定长队列一定不会向里面追加节点了，那么这个节点可以删掉了。
		// 尝试将当前的队列向后移动，因为到这可以确定，现在这个状态的tail一定是空队列了。
		if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&c.tail)), unsafe.Pointer(d), unsafe.Pointer(d2)) {
			// 移动成功以后，移动后的节点前面的节点是空队列，直接回收掉。因为head不会像前移动
			storePoolChainElt(&d2.prev, nil)
		}
		// tail是空的，已经被剔除了，d2成为新的tail，继续下一次循环。
		d = d2
	}
}

```

##### 还有一个问题，为什么popTail中d2一定是要在pop之前获取
- 考虑一个情况：这个d2在pop之后获取。如果当前这个tail的定长队列长度是8，在pop之前，这个队列是空的，且next节点是nil。然后popTail失败了，这时突然来了9个节点放进这个定长队列，那么这个定长队列是放不下的，且要生成一个长度为16的队列放在next上。这时候再获取next就会导致错误的获得了next。因为前面的队列不是空的，那么就错误的删除了前面的队列。

- 所以只有保证当前这个定长队列的next不是空的，且当前的这个定长队列是空的，才能删除当前这个定长队列
