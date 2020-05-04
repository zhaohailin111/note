coocood.freecache

**0gc的缓存，目前很多本地缓存都会有由于gc导致的程序卡顿。freecache缓存是基于ringbuf的无gc的本读缓存。用法很简单，先new再set再get**

```go
// 创建一个新的cache
func NewCache(size int) (cache *Cache) {
	if size < minBufSize { // 最小的大小是 512k
		size = minBufSize
	}
	// 每个cache有256个seg
	cache = new(Cache)
	
	// 每个seg的大小都是相同的
	for i := 0; i < segmentCount; i++ {
		cache.segments[i] = newSegment(size/segmentCount, i)
	}
	return
}
```

```go
const (
	segmentCount = 256 // seg的数量
	// 根据低8位去确定hash值对应的seg
	segmentAndOpVal = 255 // 8位
	minBufSize      = 512 * 1024
)

type Cache struct {
	locks    [segmentCount]sync.Mutex
	segments [segmentCount]segment
}
```

```go
// seg的结构
type segment struct {
	rb            RingBuf	// ringbuf是存储数据的数组
	segId         int 		// seg对应的id
	_             uint32	// 字节对齐，原子操作
	missCount     int64		// 用来统计，missCount接口
	hitCount      int64		// 用来统计，hitCount接口
	entryCount    int64		// 用来统计，entryCount接口
	
	// totalCount， totalTime lru 算法
	totalCount    int64      // 
	totalTime     int64      // 
	
	totalEvacuate int64      // used for debug
	totalExpired  int64      // used for debug
	overwrites    int64      // used for debug
	
	// 这个是ringBuf中可写入的数据大小
	vacuumLen     int64      // up to vacuumLen, new data can be written without overwriting old data.
	
	// 256个slot对应的长度和容量
	slotLens      [256]int32 // The actual length for every slot.
	slotCap       int32      // max number of entry pointers a slot can hold.
	slotsData     []entryPtr // shared by all 256 slots
}

// 创建一个seg
func newSegment(bufSize int, segId int) (seg segment) {
	// 每个seg实际上都是一个环形数组去存储数据
	seg.rb = NewRingBuf(bufSize, 0)
	
	seg.segId = segId
	
	// buf的大小
	seg.vacuumLen = int64(bufSize)
	
	// 每个seg分为256个槽，这个就是每个槽的容量
	seg.slotCap = 1
	
	// 这个是每个槽对应再ringbuf上的入口位置
	// 256个槽公用了这一个数组，槽容量在扩展的时候是256个槽一起扩展
	// 为什么用一个数组呢？
	seg.slotsData = make([]entryPtr, 256*seg.slotCap)
	return
}
```

```go
type RingBuf struct {
	// begin-end 是虚拟的数据流，也是数据写入的统计。end随着写入一直向后移动。
	begin int64 // beginning offset of the data stream.
	end   int64 // ending offset of the data stream.
	// len = end - begin
	// 实际的数据应该是 index - len(data)-1 和 0 - index 两个部分组成
	data  []byte // cap
	index int //range from '0' to 'len(rb.data)-1'
}

// 这个就没啥说的了
func NewRingBuf(size int, begin int64) (rb RingBuf) {
	rb.data = make([]byte, size)
	rb.begin = begin
	rb.end = begin
	rb.index = 0
	return
}
```

**目前为止，已经创建了制定大小的缓存。先看下ringbuf的操作都有哪些，再看看外层包装的操作**

```go
// 从off开始将数据读取到p中，返回的是读取的数量。
func (rb *RingBuf) ReadAt(p []byte, off int64) (n int, err error) {
	if off > rb.end || off < rb.begin {
		err = ErrOutOfRange
		return
	}
	
	// 这部分代码很多地方用，可以合并
	var readOff int
	if rb.end-rb.begin < int64(len(rb.data)) {
		readOff = int(off - rb.begin)
	} else {
		readOff = rb.index + int(off-rb.begin)
	}
	if readOff >= len(rb.data) {
		readOff -= len(rb.data)
	}

	readEnd := readOff + int(rb.end-off) // 数据读取的终点
	if readEnd <= len(rb.data) { // 没有涉及到环形读取
		n = copy(p, rb.data[readOff:readEnd])
	} else { // 环形读取,需要先读区末尾若干位，再从头读取剩下的
		n = copy(p, rb.data[readOff:])
		if n < len(p) {
			n += copy(p[n:], rb.data[:readEnd-len(rb.data)])
		}
	}
	if n < len(p) {
		err = io.EOF
	}
	return
}
```

```go
// 写入一部分数据
func (rb *RingBuf) Write(p []byte) (n int, err error) {
	if len(p) > len(rb.data) { // 都已经超过了ringbuf的大小了
		err = ErrOutOfRange
		return
	}
	// n默认是0，从index一直写到把p全部写入
	for n < len(p) {
		// 从index开始写入
		written := copy(rb.data[rb.index:], p[n:])
		
		// 不管是否覆盖了数据，end都会向前移动
		rb.end += int64(written)
		
		// 记录实际数据写入量，这时候n在p数组也随之移动
		n += written
		
		// index也随着written偏移
		rb.index += written
		
		// index 最多只能等于len(rb.data),这时候index，这是为0，从头开始写，循环写入
		if rb.index >= len(rb.data) {
			rb.index -= len(rb.data)
		}
	}
	// 当出现循环写入的情况，begin重新计算
	if int(rb.end-rb.begin) > len(rb.data) {
		rb.begin = rb.end - int64(len(rb.data))
	}
	return
}
```


```go
// 从off开始写入数据
func (rb *RingBuf) WriteAt(p []byte, off int64) (n int, err error) {
	// 	和write一样，检验最大长度
	if off+int64(len(p)) > rb.end || off < rb.begin {
		err = ErrOutOfRange
		return
	}
	var writeOff int
	if rb.end-rb.begin < int64(len(rb.data)) {
		writeOff = int(off - rb.begin)
	} else {
		writeOff = rb.index + int(off-rb.begin)
	}
	if writeOff > len(rb.data) {
		writeOff -= len(rb.data)
	}
	
	writeEnd := writeOff + int(rb.end-off)
	if writeEnd <= len(rb.data) {
		n = copy(rb.data[writeOff:writeEnd], p)
	} else {
		n = copy(rb.data[writeOff:], p)
		if n < len(p) {
			n += copy(rb.data[:writeEnd-len(rb.data)], p[n:])
		}
	}
	return
}

```

```go
// 从off位置开始比较
func (rb *RingBuf) EqualAt(p []byte, off int64) bool {
	if off+int64(len(p)) > rb.end || off < rb.begin {
		return false
	}
	var readOff int
	if rb.end-rb.begin < int64(len(rb.data)) {
		readOff = int(off - rb.begin)
	} else {
		readOff = rb.index + int(off-rb.begin)
	}
	if readOff >= len(rb.data) {
		readOff -= len(rb.data)
	}
	readEnd := readOff + len(p)

	if readEnd <= len(rb.data) {
		return bytes.Equal(p, rb.data[readOff:readEnd])
	} else {
		firstLen := len(rb.data) - readOff
		equal := bytes.Equal(p[:firstLen], rb.data[readOff:])
		if equal {
			secondLen := len(p) - firstLen
			equal = bytes.Equal(p[firstLen:], rb.data[:secondLen])
		}
		return equal
	}
}
```

```go
// 整个freecache的核心了
// 拷贝数据，防止覆盖
// 这个函数从逻辑的off 开始，长度length的数据，再写回到缓存中
// 等价于 rb.write(data[off:off+length])
func (rb *RingBuf)  Evacuate(off int64, length int) (newOff int64) {
	// 	off位置不对
	if off+int64(length) > rb.end || off < rb.begin {
		return -1
	}
	var readOff int // data数组，实际开始读取的位置
	// 可以不必判断?，未开始循环写入的时候，index是0
	// readOff = rb.index + int(off-rb.begin)
	if rb.end-rb.begin < int64(len(rb.data)) { // 这种情况就是数据还没有写满
		readOff = int(off - rb.begin)
	} else { // 数据已经写满了，开始循环写入
		readOff = rb.index + int(off-rb.begin) // 实际的开始位置应该是要加index的
	}

	if readOff >= len(rb.data) { // 会大于吗？
		readOff -= len(rb.data)
	}

	if readOff == rb.index { // 等价于 off == begin，不需要拷贝数据，index向后移动
		// rb.write(data[readOff:readOff+length]) 实际上，什么都没做。数据并没有任何修改，但是index却向后偏移了
		// no copy evacuate
		//if length == len(rb.data){
		//	return
		//}
		rb.index += length
		if rb.index >= len(rb.data) {
			rb.index -= len(rb.data)
		}
	} else if readOff < rb.index {
		var n = copy(rb.data[rb.index:], rb.data[readOff:readOff+length])
		rb.index += n
		// 写入的数据大于了 len(rb.data) - index, 那么还有length - n 个数据没写入，写到data中，并且index = 第二次写入的数据量
		if rb.index == len(rb.data) {
			rb.index = copy(rb.data, rb.data[readOff+n:readOff+length])
		}
	} else {
		// 这部分逻辑和write一样
		var readEnd = readOff + length
		var n int
		if readEnd <= len(rb.data) { // 还没超过data长度，直接向后写入
			n = copy(rb.data[rb.index:], rb.data[readOff:readEnd])
			rb.index += n
		} else { // 超过了，向前写入，
			n = copy(rb.data[rb.index:], rb.data[readOff:])
			rb.index += n
			var tail = length - n
			n = copy(rb.data[rb.index:], rb.data[:tail])
			rb.index += n
			if rb.index == len(rb.data) {
				rb.index = copy(rb.data, rb.data[n:tail])
			}
		}
	}
	// 重设begin end
	newOff = rb.end
	rb.end += int64(length)
	if rb.begin < rb.end-int64(len(rb.data)) {
		rb.begin = rb.end - int64(len(rb.data))
	}
	return
}
```

```go
// 舍弃一部分数据
func (rb *RingBuf) Skip(length int64) {
	rb.end += length
	rb.index += int(length)
	for rb.index >= len(rb.data) {
		rb.index -= len(rb.data)
	}
	if int(rb.end-rb.begin) > len(rb.data) {
		rb.begin = rb.end - int64(len(rb.data))
	}
}

```

**整个ringbuf有用的操作就这么多。对于seg的操作都是基于这个去做的**

```go
// 不适合，数据数量少，每个k，v很大的场景
func (seg *segment) set(key, value []byte, hashVal uint64, expireSeconds int) (err error) {
	if len(key) > 65535 { // key最大限制了
		return ErrLargeKey
	}
	
	// freecache 限制了每个key+value不能大于总的缓存值的1/1024*256
	// 分了256个slot，那么平均每个slot最少可以存储4个key， value
	maxKeyValLen := len(seg.rb.data)/4 - ENTRY_HDR_SIZE
	if len(key)+len(value) > maxKeyValLen {
		// Do not accep large entry.
		return ErrLargeEntry
	}
	now := uint32(time.Now().Unix())
	expireAt := uint32(0)
	// 过期时间
	if expireSeconds > 0 { 
		expireAt = now + uint32(expireSeconds)
	}

	// hash的低8位，用来定位seg
	// hash再顺延8位定位slot
	slotId := uint8(hashVal >> 8) 
	
	// 每个slot都是按照高16位顺序排列的
	hash16 := uint16(hashVal >> 16)

	// 	每一组kv其实都对应了一个header存储一些kv的相关信息
	var hdrBuf [ENTRY_HDR_SIZE]byte
	hdr := (*entryHdr)(unsafe.Pointer(&hdrBuf[0]))

	// 拿到slot
	slot := seg.getSlot(slotId)
	
	// 先判断是不是已经在slot里面了
	idx, match := seg.lookup(slot, hash16, key) // 判断是否匹配
	
	// 如果已经在了，就是覆盖替换
	if match {
		matchedPtr := &slot[idx] // 拿到entry
		seg.rb.ReadAt(hdrBuf[:], matchedPtr.offset) // 获取header信息
		// 更新header信息
		hdr.slotId = slotId
		hdr.hash16 = hash16
		hdr.keyLen = uint16(len(key))
		originAccessTime := hdr.accessTime
		hdr.accessTime = now
		hdr.expireAt = expireAt
		hdr.valLen = uint32(len(value))
		
		// 如果覆盖前的容量够大了，就直接在这里写进去
		if hdr.valCap >= hdr.valLen { 
			//in place overwrite
			atomic.AddInt64(&seg.totalTime, int64(hdr.accessTime)-int64(originAccessTime))
			// 写header
			seg.rb.WriteAt(hdrBuf[:], matchedPtr.offset)
			// 写data
			// 似乎浪费了空间，valCap-valLen
			seg.rb.WriteAt(value, matchedPtr.offset+ENTRY_HDR_SIZE+int64(hdr.keyLen))
			atomic.AddInt64(&seg.overwrites, 1)
			return
		}
		// avoid unnecessary memory copy.
		// 当前这个节点容量不足，删除，也是标记删除，并没有真的移动ringBuf的index
		seg.delEntryPtr(slotId, slot, idx)
		match = false
		// increase capacity and limit entry len.
		// 重设cap
		for hdr.valCap < hdr.valLen {
			hdr.valCap *= 2
		}
		// 超过最大的限制，等于最大的限制
		if hdr.valCap > uint32(maxKeyValLen-len(key)) {
			hdr.valCap = uint32(maxKeyValLen - len(key))
		}
	} else { 
		// 设置一些头部信息
		hdr.slotId = slotId
		hdr.hash16 = hash16
		hdr.keyLen = uint16(len(key))
		hdr.accessTime = now
		hdr.expireAt = expireAt
		hdr.valLen = uint32(len(value))
		hdr.valCap = uint32(len(value))
		// 因为扩容是成倍扩容的，0的话，122行会死循环
		if hdr.valCap == 0 { // avoid infinite loop when increasing capacity.
			hdr.valCap = 1
		}
	}

	// 不管上面因为key已经保存了但是容量不足，还是因为key不在缓存中，都直接当作一个新的key去写入
	// 拿到新写入的节点长度，包括header，key，value
	entryLen := ENTRY_HDR_SIZE + int64(len(key)) + int64(hdr.valCap)
	
	// 这个函数确保当前有足够的空间写入。
	slotModified := seg.evacuate(entryLen, slotId, now)
	
	// 这个是因为evacuate操作后，当前的slot可能都被删了一些数据了，那么这个slot对应的位置都变了，需要更新slot
	if slotModified {
		// the slot has been modified during evacuation, we need to looked up for the 'idx' again.
		// otherwise there would be index out of bound error.
		slot = seg.getSlot(slotId)
		// 优雅
		idx, match = seg.lookup(slot, hash16, key) // 因为走到这里，key一定不会命中了，那那么这个idx就是这个slot的新entry
		// assert(match == false)
	}
	newOff := seg.rb.End() // 新的写入位置。
	seg.insertEntryPtr(slotId, hash16, newOff, idx, hdr.keyLen)
	seg.rb.Write(hdrBuf[:])
	seg.rb.Write(key)
	seg.rb.Write(value)
	seg.rb.Skip(int64(hdr.valCap - hdr.valLen))
	atomic.AddInt64(&seg.totalTime, int64(now))
	atomic.AddInt64(&seg.totalCount, 1)
	seg.vacuumLen -= entryLen
	return
}
```

```go
// 根据slotid获取对应的slot，每个slot是若干个kv
func (seg *segment) getSlot(slotId uint8) []entryPtr {
	slotOff := int32(slotId) * seg.slotCap
	// 返回每个槽对应节点列表。第三个是容量
	return seg.slotsData[slotOff : slotOff+seg.slotLens[slotId] : slotOff+seg.slotCap]
}

```

```go
// 根据hash16和key判断key是否在这个slot中
// 返回的是匹配到的槽入口id 和 是否匹配
// 每个槽都包括很多key 和 value
func (seg *segment) lookup(slot []entryPtr, hash16 uint16, key []byte) (idx int, match bool) {
	
	// 先找到在slot里面的下标
	idx = entryPtrIdx(slot, hash16)
	for idx < len(slot) {
		ptr := &slot[idx]
		if ptr.hash16 != hash16 { // 不相同
			break
		}
		// 优化点：先判断key长度是否相同，
		// key长度相同且key值相同
		match = int(ptr.keyLen) == len(key) && seg.rb.EqualAt(key, ptr.offset+ENTRY_HDR_SIZE)
		if match {
			return
		}
		idx++ // hash冲突的处理
	}
	return
}
```

```go
// 根据hash16去定位具体的节点位置
// hash16都是有序的在slot写入的，搜索的时候可以二分
func entryPtrIdx(slot []entryPtr, hash16 uint16) (idx int) {
	high := len(slot)
	for idx < high {
		mid := (idx + high) >> 1
		oldEntry := &slot[mid]
		if oldEntry.hash16 < hash16 {
			idx = mid + 1
		} else {
			high = mid
		}
	}
	return
}
```

```go
// 在set过程中，遇到key相同且容量不够，就删掉了这个kv
// 删除slot中idx位置的kv
func (seg *segment) delEntryPtr(slotId uint8, slot []entryPtr, idx int) {
	// 这个节点的起始位置
	offset := slot[idx].offset 
	var entryHdrBuf [ENTRY_HDR_SIZE]byte // 获取头部信息
	seg.rb.ReadAt(entryHdrBuf[:], offset)
	entryHdr := (*entryHdr)(unsafe.Pointer(&entryHdrBuf[0]))
	entryHdr.deleted = true // 标记为删除然后写进去
	seg.rb.WriteAt(entryHdrBuf[:], offset)
	
	// 这里是向前移动了，将这个kv在当前的slot删掉了，但是在ringbuf里面并没有删掉。
	copy(slot[idx:], slot[idx+1:])
	seg.slotLens[slotId]--
	atomic.AddInt64(&seg.entryCount, -1)
}
```

```go
// 向slot中插入一个kv
func (seg *segment) insertEntryPtr(slotId uint8, hash16 uint16, offset int64, idx int, keyLen uint16) {
	// slot已经满了，整体扩容。
	if seg.slotLens[slotId] == seg.slotCap {
		seg.expand()
	}
	// 这个slot长度加一
	seg.slotLens[slotId]++
	atomic.AddInt64(&seg.entryCount, 1)
	slot := seg.getSlot(slotId)
	copy(slot[idx+1:], slot[idx:]) // 向后移动1位。因为所有slot公用一个slot数组了
	slot[idx].offset = offset
	slot[idx].hash16 = hash16
	slot[idx].keyLen = keyLen
}
```

```go
// 扩容，某个slot不够用来，将slot翻一倍
// 扩容是将256个slot都翻倍
func (seg *segment) expand() {
	newSlotData := make([]entryPtr, seg.slotCap*2*256)
	for i := 0; i < 256; i++ {
		off := int32(i) * seg.slotCap
		copy(newSlotData[off*2:], seg.slotsData[off:off+seg.slotLens[i]])
	}
	seg.slotCap *= 2
	seg.slotsData = newSlotData
}
```

```go
// 相应的，这个也是seg的核心了
// evacuate过程是这样的：
// 1、拿到ringbuf中的第一个kv
// 2、如果kv是已经过期的，或者因为lru淘汰的，或者因为循环次数过多随机淘汰的，那就把它删除
// 3、如果不满足上述条件，就把这个kv放到整个ringbuf的最前方。这样才能保证有一段连续的足够大空间存储数据
func (seg *segment) evacuate(entryLen int64, slotId uint8, now uint32) (slotModified bool) {
	var oldHdrBuf [ENTRY_HDR_SIZE]byte
	consecutiveEvacuate := 0
	for seg.vacuumLen < entryLen { // 剩余空间不足
		// 如果数据写满了，那么oldOff = begin，可以写入
		// 如果没写满 oldOff 等于0
		// oldOff == begin？？
		oldOff := seg.rb.End() + seg.vacuumLen - seg.rb.Size()
		seg.rb.ReadAt(oldHdrBuf[:], oldOff)
		oldHdr := (*entryHdr)(unsafe.Pointer(&oldHdrBuf[0]))
		oldEntryLen := ENTRY_HDR_SIZE + int64(oldHdr.keyLen) + int64(oldHdr.valCap)
		if oldHdr.deleted { // 如果已经删除了
			consecutiveEvacuate = 0
			atomic.AddInt64(&seg.totalTime, -int64(oldHdr.accessTime))
			atomic.AddInt64(&seg.totalCount, -1)
			seg.vacuumLen += oldEntryLen
			continue
		}
		expired := oldHdr.expireAt != 0 && oldHdr.expireAt < now // 是否过期
		// 淘汰算法 lru 访问时间小于平均访问时间
		leastRecentUsed := int64(oldHdr.accessTime)*atomic.LoadInt64(&seg.totalCount) <= atomic.LoadInt64(&seg.totalTime)
		// 应该淘汰这个数据了
		// consecutiveEvacuate连续Evacuate 6个entry，就会删掉这个entry了
		// consecutiveEvacuate > 5 表明这个slot里面都是热点的key, 会淘汰第7个
		// 为啥这个地方是>5
		if expired || leastRecentUsed || consecutiveEvacuate > 5 {
			seg.delEntryPtrByOffset(oldHdr.slotId, oldHdr.hash16, oldOff) // 调这个接口，这个槽中的entry位置都变了
			if oldHdr.slotId == slotId { // 淘汰旧的数据
				slotModified = true
			}
			consecutiveEvacuate = 0
			atomic.AddInt64(&seg.totalTime, -int64(oldHdr.accessTime))
			atomic.AddInt64(&seg.totalCount, -1)
			seg.vacuumLen += oldEntryLen // 这里增加
			if expired {
				atomic.AddInt64(&seg.totalExpired, 1)
			}
		} else {
			// evacuate an old entry that has been accessed recently for better cache hit rate.
			// 数据转移到最新的位置，也就是ringbuf的最前面了，刚才说了ringbuf的Evacuate就是把数据拿出来再写进去。
			newOff := seg.rb.Evacuate(oldOff, int(oldEntryLen)) 
			seg.updateEntryPtr(oldHdr.slotId, oldHdr.hash16, oldOff, newOff) // 更新这个entry的offset
			consecutiveEvacuate++
			atomic.AddInt64(&seg.totalEvacuate, 1)
		}
	}
	return
}
```

```go
// 相应的在evacuate过程中，遇到过期的kv，也是要删除
// 根据offset删除slot中的key value
// 删除是标记删除
func (seg *segment) delEntryPtrByOffset(slotId uint8, hash16 uint16, offset int64) {
	slot := seg.getSlot(slotId)
	
	// 根据offset找到这个kv，然后删掉
	idx, match := seg.lookupByOff(slot, hash16, offset)
	if !match {
		return
	}
	seg.delEntryPtr(slotId, slot, idx)
}
```

```go
// 根据偏移量判断是否在slot中
// 和lookup差不了多少，只是在判断的时候一个判断key是不是相同，一个判断off是不是相同
func (seg *segment) lookupByOff(slot []entryPtr, hash16 uint16, offset int64) (idx int, match bool) {
	idx = entryPtrIdx(slot, hash16)
	for idx < len(slot) {
		ptr := &slot[idx]
		if ptr.hash16 != hash16 {
			break
		}
		match = ptr.offset == offset
		if match {
			return
		}
		idx++
	}
	return
}
```

```go
// 在evacuate过程中，需要把数据不断的挪到ringbuf的前面，这样保证能够有足够长的一段连续的空间去存储数据。
// 更新entry在ringBuf中的offset
func (seg *segment) updateEntryPtr(slotId uint8, hash16 uint16, oldOff, newOff int64) {
	slot := seg.getSlot(slotId)
	idx, match := seg.lookupByOff(slot, hash16, oldOff)
	if !match {
		return
	}
	ptr := &slot[idx]
	ptr.offset = newOff
}
```


```go
// 从seg中获取key对应的value
func (seg *segment) get(key, buf []byte, hashVal uint64) (value []byte, expireAt uint32, err error) {
	// 还是一样
	slotId := uint8(hashVal >> 8)
	hash16 := uint16(hashVal >> 16)
	slot := seg.getSlot(slotId)
	idx, match := seg.lookup(slot, hash16, key) // 获取槽入口id和是否匹配
	if !match { // 不匹配直接返回了
		err = ErrNotFound
		atomic.AddInt64(&seg.missCount, 1)
		return
	}
	ptr := &slot[idx] // 入口节点
	now := uint32(time.Now().Unix())

	var hdrBuf [ENTRY_HDR_SIZE]byte
	seg.rb.ReadAt(hdrBuf[:], ptr.offset)
	hdr := (*entryHdr)(unsafe.Pointer(&hdrBuf[0])) // 获取键值对的header
	expireAt = hdr.expireAt
	
	// 匹配还是要判断是不是过期了
	if hdr.expireAt != 0 && hdr.expireAt <= now { 
		seg.delEntryPtr(slotId, slot, idx) // 删除这个slot中的entry
		atomic.AddInt64(&seg.totalExpired, 1)
		err = ErrNotFound
		atomic.AddInt64(&seg.missCount, 1)
		return
	}
	atomic.AddInt64(&seg.totalTime, int64(now-hdr.accessTime))
	hdr.accessTime = now // lru
	seg.rb.WriteAt(hdrBuf[:], ptr.offset)
	if cap(buf) >= int(hdr.valLen) {
		value = buf[:hdr.valLen]
	} else {
		value = make([]byte, hdr.valLen)
	}

	seg.rb.ReadAt(value, ptr.offset+ENTRY_HDR_SIZE+int64(hdr.keyLen)) // 读取value
	atomic.AddInt64(&seg.hitCount, 1)
	return
}
```

```go
// 删除key
func (seg *segment) del(key []byte, hashVal uint64) (affected bool) {
	slotId := uint8(hashVal >> 8)
	hash16 := uint16(hashVal >> 16)
	slot := seg.getSlot(slotId)
	idx, match := seg.lookup(slot, hash16, key)
	if !match {
		return false
	}
	seg.delEntryPtr(slotId, slot, idx)
	return true
}
```

```go
// 查询剩余过期时间
func (seg *segment) ttl(key []byte, hashVal uint64) (timeLeft uint32, err error) {
	// 这里很多地方在用
	slotId := uint8(hashVal >> 8)
	hash16 := uint16(hashVal >> 16)
	slot := seg.getSlot(slotId)
	idx, match := seg.lookup(slot, hash16, key)
	if !match {
		err = ErrNotFound
		return
	}
	ptr := &slot[idx]
	now := uint32(time.Now().Unix())

	// 获取key value 的头部信息
	var hdrBuf [ENTRY_HDR_SIZE]byte
	seg.rb.ReadAt(hdrBuf[:], ptr.offset)
	hdr := (*entryHdr)(unsafe.Pointer(&hdrBuf[0]))

	// 没有设置过期时间，为什么不返回-1呢
	if hdr.expireAt == 0 {
		timeLeft = 0
		return
	} else if hdr.expireAt != 0 && hdr.expireAt >= now {
		timeLeft = hdr.expireAt - now // 获取剩余时间
		return
	}
	// 剩下的就是已过期的情况了，返回未找到
	err = ErrNotFound
	return
}
```

**cache操作就是简单的hash，加锁了**

**整体非常清爽，先申请，再维护而不是动态的，让我想起了刷题的时候写字典树**

**很多地方看起来都是比较耗时的，包括lru算法总是去拿header再set，挪动数组保证足够大的连续空间，slot也是copy来copy去。但实际应用的时候性能还是非常好的。毕竟内存操作相对io操作来说耗时是很低的**
