# sync.Map

```go
// 当value值为这个的时候，表示该节点已经被删除了，但是这个entry还没有从内存中删掉
// 普通的map最大的问题就是并行扩容和读
// sync.Map的核心是两个map，一个是只读map，一个非只读的map
// 这个只读的map不是key只读。而是不能扩容和缩容。
// 当一个key从只读中删掉后，这个key对应的value就是nil
// 在删除的时候，如果这个key在只读和非只读都有的话，则将value变成nil
// 在读取的时候，先走只读，miss后走非只读。如果miss次数太多，只读直接被非只读替换。非只读直接变成nil
// 在存储的时候，出现一个新的key就直接如果非只读是nil，而只读不能扩容。这时候将全量的数据从只读同步到非只读，那些nil的key就直接被标记为expunged。表示这个key已经被删除了，但是在只读中还有这个key，只是没有对应的value，而非只读的中已经没有这个key了。
// 而只读的map就是read，非只读的就是dirty

// 总结来说，这个状态就是表示：read中为空而dirty中没有的key。
var expunged = unsafe.Pointer(new(interface{}))

type Map struct {
	// 互斥锁，当操作dirty会先获取锁
	mu Mutex
	
	// 只读map，不可扩容。
	// 每次有新的key都不会在read中存储
	// 每次删除key，也不会真的从read中删除
	read atomic.Value // readOnly

	// dirty 并不是真正的map。
	// 只是有新的key存储的时候，在dirty中存下
	// 当dirty中有read中没有的key的时候，dirty才是真正的map
	dirty map[interface{}]*entry

	// 当从dirty中获取数据，miss++
	// 当miss次数和dirty中key一样，则用dirty替换read
	misses int
}

// 一个只读的map包括一个map和一个标记
// 该标记为真表示，dirty中包括read中没有的key
type readOnly struct {
	m       map[interface{}]*entry
	amended bool 
}

// entry是在map中具体存储value的地址
type entry struct {
	// 当p为nil的时候，表示这个被删了，但是只是被标记删除了
	// 当p为expunged的时候，表示这个key被删了，但是在read中还有这个key，而dirty中没了
	// 当p是nil或者expunged，都表示这个key被删除了。
	p unsafe.Pointer // *interface{}
}

// 生成一个新的entry，p为传入的i的地址
// 返回的也是这个entry的地址，说明不管是read还是dirty存储的都是地址
// 也就是更新了read中的，dirty也随之更新了
func newEntry(i interface{}) *entry {
	return &entry{p: unsafe.Pointer(&i)}
}

// 从map中获取一个key
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	// 先从只读map中获取
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	// 如果read中没有，并且只读dirty中有read中没有的key
	if !ok && read.amended {
		m.mu.Lock()
		// 加锁之前，map可能发生了变化。重新获取
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		
		// 如果之前判断的条件不再满足了，那么e和ok已经被更新
		// 进行后续操作也没有问题了
		if !ok && read.amended {
			// 从dirty中获取这个entry
			// 再次更新e和ok
			e, ok = m.dirty[key]
			
			// 从dirty中获取key的时候都要判断是否满足替换条件
			m.missLocked()
		}
		m.mu.Unlock()
	}
	
	// ok可能是从只读map中得到的
	// 也可能从dirty中得到的。
	// 但是到这里一定是只读和dirty中都没有了
	// 这个key既不在只读也不在dirty，返回false
	if !ok {
		return nil, false
	}
	// 不管是从只读还是dirty中获取的entry，都load返回
	return e.load()
}

// load时如果尝试从dirty中获取key
// 也就是只读中没有但是dirty中有，但是不一定在dirty中，则计数加1
// 当计数值和dirty中的key相同时，替换只读map
// 用来减少锁竞争，操作dirty需要加锁
func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}
	
	// 次数太多了，将dirty放进readOnly并存到read
	m.read.Store(readOnly{m: m.dirty})
	
	// 清空dirty，重置计数器
	m.dirty = nil
	m.misses = 0
}

// 从当前的这个entry获取value
func (e *entry) load() (value interface{}, ok bool) {
	// 获取p
	p := atomic.LoadPointer(&e.p)
	// p被删除了，返回false
	if p == nil || p == expunged {
		return nil, false
	}
	// 将p转化为空接口，返回true
	return *(*interface{})(p), true
}

// 将key存入map
func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly)
	
	// 从read中获取entry，如果在read中。尝试更新
	// 如果更新value成功了，直接返回
	// 尝试更新的时候如果entry已经被从dirty删了，则返回失败
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	// 三种情况
	// 1、在read，不在dirty
	// 2、不在read，在dirty
	// 3、都不在
	
	// 需要操作dirty
	m.mu.Lock()
	// 从只读map中再读一次，因为此时map可能已经被修改
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		// 在read，不在dirty
		// 将entry修改为nil，因为此时状态是expunged
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		// 存储value
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
		// 在dirty不在read，直接更新，也不会存储到read
		e.storeLocked(&value)
	} else {
		// 该节点是全新的，不在dirty也不在read
		// 此时两种情况：dirty和read中的数量一样/不一样
		if !read.amended {
			// 如果一样，可能需要将整个dirty数据都重新load，dirty可能还是空的
			m.dirtyLocked()
			// 整个替换read
			// 修改状态为不一样，因为read没有变化。这个key导致dirty中有read中没有的key
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		// 在dirty中存储一个新节点，在read中没有这个节点
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}

// 如果当前的entry是expunged，不能更新，因为dirty中没有这个entry
// 其他情况为空或者非空都直接存储，这时候dirty也随之更新了
func (e *entry) tryStore(i *interface{}) bool {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
			return true
		}
	}
}

// 修改状态expunged为nil。这时候是有一个key不在dirty中，但是需要存储他。所以修改状态
func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// 将value存储到entry
func (e *entry) storeLocked(i *interface{}) {
	atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

// 初始化dirty
// map首次建立的时候或者read被dirty替换的时候都会进行这个操作
func (m *Map) dirtyLocked() {
	// 如果dirty不为空，直接返回
	if m.dirty != nil {
		return
	}
	
	
	// 将read家在到dirty中
	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
	for k, e := range read.m {
		// 对于每个key，需要判断这个key是否被删除了，如果被删除了，那么read中就是nil
		// 相应的，在dirty中改为expunged
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}

// 尝试将节点修改为expunged状态
// 如果节点为空，则改为expunged，返回成功
// 如果节点为expunged，则返回成功。
// 如果节点不为空，则返回修改失败
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}


// map中删除
func (m *Map) Delete(key interface{}) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	
	// 如果不在read且dirty中有read没有的，从dirty删除
	if !ok && read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			delete(m.dirty, key)
		}
		m.mu.Unlock()
	}
	if ok {
		// 如果在read中，则标记状态为nil
		e.delete()
	}
}

// 删除某个节点，将节点改为nil
func (e *entry) delete() (hadValue bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		// 如果节点为空或expunged
		if p == nil || p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return true
		}
	}
}

// 节点不为空则将i存入，否则取出该节点
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
	// 先看read
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		// 如果read中有，操作
		actual, loaded, ok := e.tryLoadOrStore(value)
		// !ok是因为节点从dirty中删了，但是read中还有
		if ok {
			return actual, loaded
		}
	}
	// 操作失败 也是三种情况
	// 1、read中有，dirty删了。
	// 2、read中没有，dirty中有
	// 3、都没有
	
	// 和store一样，只是对应的都改成了LoadOrStore
	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		actual, loaded, _ = e.tryLoadOrStore(value)
	} else if e, ok := m.dirty[key]; ok {
		actual, loaded, _ = e.tryLoadOrStore(value)
		m.missLocked()
	} else {
		if !read.amended {
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
		actual, loaded = value, false
	}
	m.mu.Unlock()

	return actual, loaded
}

// 第一个返回值是load的结果
// 第二个返回值是是否load成功，如果节点为空，则直接store，此时这个值是false
// 第三个返回值是是否操作成功，如果节点是expunged，说明已经从dirty删了，这个值是false
func (e *entry) tryLoadOrStore(i interface{}) (actual interface{}, loaded, ok bool) {
	p := atomic.LoadPointer(&e.p)
	// 如果p是expunged，返回
	if p == expunged {
		return nil, false, false
	}
	// 如果p不为空，直接返回该值。
	if p != nil {
		return *(*interface{})(p), true, true
	}

	// 此时p==nil
	// 尝试将i存储节点
	ic := i
	for {
		// 原子修改，成功则返回
		if atomic.CompareAndSwapPointer(&e.p, nil, unsafe.Pointer(&ic)) {
			return i, false, true
		}
		// 走上述流程
		//这是个do while。。。
		p = atomic.LoadPointer(&e.p)
		if p == expunged {
			return nil, false, false
		}
		if p != nil {
			return *(*interface{})(p), true, true
		}
	}
}

// 遍历
// range传入一个函数，接收kv两个参数，返回一个bool值。当执行函数返回false，遍历终止。
func (m *Map) Range(f func(key, value interface{}) bool) {
	// 先读取read。如果dirty中包含read中没有的key。使用dirty覆盖read。
	read, _ := m.read.Load().(readOnly)
	if read.amended {
		// m.dirty contains keys not in read.m. Fortunately, Range is already O(N)
		// (assuming the caller does not break out early), so a call to Range
		// amortizes an entire copy of the map: we can promote the dirty copy
		// immediately!
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		if read.amended {
			read = readOnly{m: m.dirty}
			m.read.Store(read)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}
	// 保证read是最新的
	// 最终循环遍历read。
	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		if !f(k, v) {
			break
		}
	}
}
```
