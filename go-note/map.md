# map
1.14.2

**拉链法**

**对于hash以后的结果，如果后缀相同的，放在一个桶里面，也就是拉出一个链表，桶的大小是8。如果拉出的链表大于8会生成一个溢出桶。溢出桶会决定什么时候扩容**

**所有操作都是无锁的，map不能并发读写**

**扩容触发依赖总数量和溢出桶数量**

- 总数量过多会触发扩容
- 溢出桶数量过多会触发扩容（防止出现总数量不多，但是hash算法不良导致都跑到一个桶里面）

```go
// 常量和结构
const (

	// 每个桶最多元素数量
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits

	// 溢出因子13/2=6.5
	// 总数量/桶数量 > 6.5 需要扩容
	loadFactorNum = 13
	loadFactorDen = 2

	maxKeySize  = 128
	maxElemSize = 128

	// 每个桶都保存了头部的信息
	// hash桶的结构就是头部信息+8ks+8vs+指向下一个桶指针
	// 加一个int64为了子节对齐
	dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)

	// map中每个元素的状态
	emptyRest      = 0 // 当前是空的，并且当前这个元素后面也都是空的了
	emptyOne       = 1 // 当前是空的，后面有可用的元素
	evacuatedX     = 2 // 扩容时转移到低位桶
	evacuatedY     = 3 // 扩容时转移到高位桶
	evacuatedEmpty = 4 // 扩容的时候元素是空的
	minTopHash     = 5 // hash桶头部存了tophash加速查找，0-4用来存状态。这个是分界线

	// map的状态
	iterator     = 1 // there may be an iterator using buckets
	oldIterator  = 2 // there may be an iterator using oldbuckets
	hashWriting  = 4 // a goroutine is writing to the map
	sameSizeGrow = 8 // 正在等量扩容

	// sentinel bucket ID for iterator checks
	noCheck = 1<<(8*sys.PtrSize) - 1
)

// A header for a Go map.
type hmap struct {
	count     int // map中元素数量
	flags     uint8 // map当前的状态
	B         uint8  // map中桶数量
	noverflow uint16 // 记录有多少溢出桶
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // 存储数据的桶s
	oldbuckets unsafe.Pointer // 用来扩容的桶s
	nevacuate  uintptr   // 扩容的时候，旧桶迁移的进度。当此值等于len(oldbuckets)，迁移完成      

	extra *mapextra // optional fields
}
```
```go
// 工具函数

// 2^b
// bucketShift returns 1<<b, optimized for code generation.
func bucketShift(b uint8) uintptr {
	// Masking the shift amount allows overflow checks to be elided.
	// todo？
	return uintptr(1) << (b & (sys.PtrSize*8 - 1))
}

// 2^b-1
// bucketMask returns 1<<b - 1, optimized for code generation.
func bucketMask(b uint8) uintptr {
	return bucketShift(b) - 1
}

// 取hash值的高8位
func tophash(hash uintptr) uint8 {
	// 这个top是计算出来的hash右移56位
	// 如果是int64位，则最高的8个01串就是最终结果
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	
	// wc，为了不额外使用变量存储状态
	// tophash占用了5个数去进行标记这个位置元素的状态
	// 那进行比较的tophash就要大于5，如果小于5，则表示的hash状态
	if top < minTopHash {
		top += minTopHash
	}
	return top
}

// 是否溢出，需要扩容
func overLoadFactor(count int, B uint8) bool {
	return 
		// 总数还不到一个桶，肯定不需要扩容
		count > bucketCnt && 
		
		// 总数/桶数量 > 13/2 溢出因子就是6.5
		uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

```go
// 创建一个新的map，比较简单
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// 计算合适的桶的数量
	// 计算出来的因子大于6.5就会扩容
	// 元素数量 / 桶数量 > 6.5
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// 分配对应的桶，和溢出桶。
	// map的每个桶只有8个元素，超过了就会进入溢出桶
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

// 根据桶大小获取桶和溢出桶
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	// 桶的数量
	base := bucketShift(b)
	nbuckets := base
	
	// 需要提前预估桶数量
	// 如果普通桶数量小于2^4。那就不太可能有溢出桶
	if b >= 4 {
	
		// 预分配的桶数量是2^(b-4)
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	// dirtyalloc是否有脏的空间
	// 如果没有的话说明是扩容，或者创建
	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
		
	// clear这个map
	} else {
		// 因为dirtyalloc不是空的所以直接复用空间
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		// todo，后面这两个函数
		// 根据桶中指针类型变量数量判断堆还是栈
		if t.bucket.ptrdata != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	// 说明有溢出桶
	if base != nbuckets {
		// 第一个溢出桶就是直接放在正常桶的后面
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		
		// 最后一个溢出桶后面又跟着正常的桶
		// 为了给最后一个桶一个安全指针
		// 后面根据这个指针是否为空来判断是否是最后一个溢出桶
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

```go
// map查找元素
// a := map[b]
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 
	if h == nil || h.count == 0 {
		// 这个大概意思是，如果key使用了不能比较的类型。可能会panic？
		// 实际使用的时候就是这个key如果不能hash就会出现panic
		// 这个不能hash就是key必须由基本类型组合而成的，比如一个结构体只包含基本类型和只包含基本类型的结构体
		if t.hashMightPanic() { 
			t.hasher(key, 0) // see issue 23734，这里可以看到具体的例子
		}
		return unsafe.Pointer(&zeroVal[0])
	}
	
	// 同时读写map
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	
	// 计算hash值
	hash := t.hasher(key, uintptr(h.hash0))
	
	// 2^B - 1 
	m := bucketMask(h.B)
	
	// 根据m获取对应的桶，
	// h.buckets是桶数组的起始地址。hash&m使用hash低h.B位找到对应的桶。
	// 比如，有2^3个桶，桶编号分别是000, 001, 010, 011, 100, 101, 110, 111 8个桶
	// 如果hash值是100101……1011，那么对应的桶就是011这个桶。
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	
	// 如果map正在扩容
	if c := h.oldbuckets; c != nil {
	
		// 判断扩容后桶数量是否翻倍
		if !h.sameSizeGrow() {
			m >>= 1 // 每次扩容都是*2，那么旧桶数量就是m/2,向下取整。
		}
		
		// 旧桶，如果旧的桶还没有迁移到新的桶，就去旧桶里面找。
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		
		// 判断旧桶中第一个元素的tophash。
		// 如果处于扩容，会将桶重新分配。
		// 如果第一个tophash是有效的，说明当前的k对应的桶还没有转移
		if !evacuated(oldb) {
			b = oldb
		}
	}
	
	// 计算tophash值，这个用来在桶里面加速的。如果tophash不同，那么肯定不是对应的key
	top := tophash(hash)
bucketloop:

	// 最外层循环是循环的溢出桶，如果在b这个桶里面没找到，那么要在全部的溢出桶中去找。
	for ; b != nil; b = b.overflow(t) {
	
		// 这个循环是循环桶中的8个元素
		for i := uintptr(0); i < bucketCnt; i++ {
			
			// 拿tophash比较。
			if b.tophash[i] != top {
			
				// 如果比较的是不相同的，且是个空的hash，那么说明桶已经遍历完了。直接结束
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				
				// 否则的话比较下一个元素
				continue
			}
			
			// 拿到对应的key
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			
			// 将地址转化为值
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			
			// 判断拿到的key是不是和传入的相同
			// 拿到的是tophash相同的k，再次比较
			if t.key.equal(key, k) {
			
				// 计算对应value的地址
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				
				// 返回value的值
				return e
			}
		}
	}
	
	// 默认值
	return unsafe.Pointer(&zeroVal[0])
}

mapaccess2 基本上是一样的，在返回value的时候多返回一个true。
mapaccessK 也是，只是将key也返回了
```

```go
// 写入元素
// 当map[k]在左边的时候，返回的地址然后赋值
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 不能向空map写入,但是可以从空map读出默认值
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	
	// 并发写的问题
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	
	hash := t.hasher(key, uintptr(h.hash0))

	// ？为啥不直接赋值
	// 修改当前的读写状态那个
	h.flags ^= hashWriting

	// 当前桶是空的，分配新桶，map在创建的时候可能没分配。
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
	// 获取当前的桶
	bucket := hash & bucketMask(h.B)
	
	// 如果当前的桶在扩容
	if h.growing() {
	
		// 执行扩容操作 todo
		growWork(t, h, bucket)
	}
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:

	// 最外层的死循环和读取是一样的，溢出桶
	for {
	
		// 对于当前桶的8个元素
		for i := uintptr(0); i < bucketCnt; i++ {
		
			// 如果当前元素和tophash不同
			if b.tophash[i] != top {
				
				// 如果当前的元素是空的且首次循环
				// 拿到这个元素的k和v的地址
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				
				// 这里怎么理解？
				// emptyRest表示当前的位置是空的，当前的桶后面的位置以及溢出桶都是空的了
				// 这个时候不需要继续遍历当前桶和溢出桶了
				// 这时候只存在一种情况，就是上面的if拿到了一个空元素
				// 那么为什么不在上面break呢
				// 因为可能还存在和当前key等值的情况，上面的if只是找到一个空的可写元素
				// 如果上面break了，那么就可能存在两个相同的key了
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			// 上述tophash不同的情况要么继续，要么break。这里一定是相同了
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			
			// 如果key和拿出来的k不同，说明只是tophash一样，直接看下一个元素
			if !t.key.equal(key, k) {
				continue
			}
			
			// 当前的key需要更新
			// todo
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			
			// 获取这个地址
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			// 可以返回了
			goto done
		}
		// 桶都遍历完了，去下一个溢出桶看看
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	// 什么时候能走到这里
	// 当前的要赋值的key不在这个map里面
	// 1、一定是找到了一个可以访问的空的元素地址
	// 2、当前的桶已经满了，而且没有可用的溢出桶了
	
	// 如果当前没在扩容，并且超过了因子或者溢出桶太多了
	// 那就扩容，然后重新再找一次
	// 也有可能一致没找到，那就一直执行到扩容完成
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	// 上面的第二个情况，没有可以放入的桶了
	if inserti == nil {
		// 创建一个新的桶，并且第一个元素就是要写入的地方
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// 写入当前指定的k和v里面
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	
	// 修改map写入状态
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}

// map的代码也是充斥的goto还有循环，但是流程上还是很清晰的

// 新建一个溢出桶
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
	var ovf *bmap
	// 在新分配map的时候根据桶数量会额外分配一些溢出桶
	// 这里判断一下是不是有溢出桶
	if h.extra != nil && h.extra.nextOverflow != nil {
		
		// 如果有溢出桶的话，那就拿来一个溢出桶
		ovf = h.extra.nextOverflow
		
		// 创建溢出桶的时候，将最后一个桶指向一个非空指针
		// 这里根据这个指针是否为空来判断是不是最后一个
		if ovf.overflow(t) == nil {
			// 如果不是最后一个，那就把让下一个变成可用溢出桶
			h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
		} else {
		
			// 如果是最后一个了，那就把下一个可用溢出桶变为空
			// 新拿的溢出桶因为在初始化的时候指向了非空，这里重新置为空
			ovf.setoverflow(t, nil)
			h.extra.nextOverflow = nil
		}
	} else {
	
		// 没有可用溢出用，新建一个
		ovf = (*bmap)(newobject(t.bucket))
	}
	
	// 增加溢出桶数量
	// 当桶数量较少，这个是精确的
	// 当桶数量较多，这个是近似值
	h.incrnoverflow()
	
	// 不包含指针就使用这个切片？todo
	if t.bucket.ptrdata == 0 {
		h.createOverflow()
		*h.extra.overflow = append(*h.extra.overflow, ovf)
	}
	
	// 当前这个b的溢出桶指向拿来的这个
	b.setoverflow(t, ovf)
	return ovf
}
// 到这可以看到，每个桶都是一个链表了。预先分配了一些正常桶和溢出桶。
// 正常桶满了，再写数据就会从溢出桶拿来一个接在正常桶后面


// 扩容
// 扩容分为两个模式，也就是上面说的两种触发情况
// 1、桶数量少，总数太多，这时候需要增加桶数量
// 2、桶数量足够，但是溢出太多了。
// 可以看到扩容的时候不是一次扩容，而是进行一些初始化。
// 真正扩容的时候是每次growWork
func hashGrow(t *maptype, h *hmap) {
	bigger := uint8(1)
	
	// 判断溢出因子是不是过大来决定是不是要增加桶数量
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	// bigger=1，说明桶数量加倍
	
	// 分配新的桶和溢出桶
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}

// 扩容工作流程
// 触发时机：删除或者写入的时候判断是否在扩容。
// 扩容期间如果要再次扩容，则循环执行这个直到扩容完成
// 其实可以看到读不会触发扩容，只会选择在新的还是旧的桶去找
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// 扩容当前的桶
	evacuate(t, h, bucket&h.oldbucketmask())

	// h.nevacuate表示扩容的进度，如果当前还在扩容，则再扩容一个
	// 为什么再判断一次，因为上面扩容的桶可能就是最后一个了
	// 为什么多扩容一个，防止扩容无休止的进行下去
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}

// 扩容是按照桶维度的，可以看到传入的是旧的桶地址。每次也是扩容一个桶
// 扩容分为等量和翻倍两种
// 等量扩容是将hash里面的所有元素都合并在一起。
// 因为删除只是标记删除，桶的位置还是占有的，
// 所以需要把所有为空的元素真正的删除，这样就有足够的桶或者溢出桶使用了。
// 有一种情况，如果当前桶已经写完了，并且没有溢出桶可用了怎么办。
// 可以看到在写入的时候会循环一直扩容直到可以写入了。
// 翻倍则是把当前桶分到两个不同的桶里面
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	// 拿到对应的桶
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	
	// 旧桶的数量，新扩容的桶数量
	newbit := h.noldbuckets()
	
	// 循环判断当前桶是否扩容完成
	if !evacuated(b) {
		// 扩容分为翻倍扩容和等量扩容两种情况
		// 如果是翻倍扩容，扩容以后就会将数据写在xy数组两个桶里面
		// 如果是等量扩容，那只会使用xy[0]
		var xy [2]evacDst
		
		// 不管什么扩容都初始化第一个桶
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))

		// 如果不是等量扩容，还要初始化第二个桶
		if !h.sameSizeGrow() {
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}
		
		// 循环遍历当前桶以及其溢出桶
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
			
			// 依次遍历桶中的k和v
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
			
				// 拿到这个元素的状态
				// 如果这个元素是空的，则设置为扩容后为空
				// 这个状态for range map会用到
				// 这个状态是否有必要呢？
				// 迭代的时候也不会访问他
				top := b.tophash[i]
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				
				// 并发扩容，上面判断是否为空了
				if top < minTopHash {
					throw("bad map state")
				}
				
				// 拿到当前的key
				k2 := k
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				
				// 判断是否将当前元素写进xy[1]里面
				// 如果是等量扩容，这个值就是0，
				// 如果翻倍扩容，则hash判断是否为1
				var useY uint8
				if !h.sameSizeGrow() {
					hash := t.hasher(k2, uintptr(h.hash0))
					
					// 判断useY是否为1
					// 如果此时在迭代
					// 并且这个k不是NAN则if成立
					// 这时候根据tophash最低位判断用哪个xy，并且还要重算top，因为hash变了
					// 去了哪里其实是完全随机的了
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
						// 根据旧的top决定是否在y桶
						// 新的hash再计算top
						// 这个top本质上也没什么意义了。
						// 这种k不能查找更新删除
						// 唯一要考虑的就是迭代的时候要迭代它
						useY = top & 1
						top = tophash(hash)
					} else {
						
						// 扩容的时候是翻倍的，那么低于oldn（旧桶数量）则是useY=0
						// 高于oldn（旧桶数量） 则是useY=1
						// 根据hash的后（桶数量）位判断是在哪个桶。
						// 如果就桶是8个，则&7就是对应的桶了。
						// 那如果变成了16个，那就需要&15
						// 而newbit值此时是8，&15其实就是等于就桶的位置&8也就是&7&8
						// 所以这里只判断最高位就可以知道是在高位还是低位了。
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}
		
				// 修改扩容状态，是在useX还是useY
				b.tophash[i] = evacuatedX + useY				// 拿到当前桶扩容以后对应的桶
				dst := &xy[useY]

				// 这里判断扩容以后的桶是否溢出了，溢出了就放在扩容后的溢出桶里面
				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				
				// 设置对应的tophash
				dst.b.tophash[dst.i&(bucketCnt-1)] = top 
				
				// 设置对应的k和v
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
				
				// 游标移动到下一个位置
				dst.i++
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
		
		// 如果元素包含指针类型，删除指针引用？
		// 这个用来gc的
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	// 扩容终止判断
	// 因为扩容会扩当前桶并且根据游标再扩一个
	// 如果是扩了最后一个，需要判断是否终止
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}

func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	// 当前这个桶已经被扩容了，先加一
	h.nevacuate++
	// 依次向后遍历1024个。为啥是1024个呢
	// 因为有的桶可能已经转移了，所以这里进行一个加速
	// 如：共2048个桶，依次访问了1025-2048的桶，那这些桶也都被迁移了
	// 与此同时，每次迁移还要从0开始额外迁移。所以1-1024也被迁移了
	// 当迁移到1024桶时，发现1025已经迁移了，那迁移游标也要增加
	stop := h.nevacuate + 1024
	
	// 不能超过桶数量
	if stop > newbit {
		stop = newbit
	}
	
	// 游标循环增加至指向一个未被迁移的桶
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
	
	// 如果游标增加到最后一个了，说明整个map扩容完毕，释放。旧的桶
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
		h.oldbuckets = nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}
```

```go
// map删除元素
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}

	hash := t.hasher(key, uintptr(h.hash0))

	// 设置map状态
	h.flags ^= hashWriting

	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	
	// 把当前的桶保存下，在删除的时候可能b就变成溢出桶了
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
		
			// tophash不同直接继续
			if b.tophash[i] != top {
			
				// 这里判断当前桶后面没了，并且溢出桶没了
				// 直接返回，map里面没有这个key
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			
			// 拿到k并比较
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !t.key.equal(key, k2) {
				continue
			}
			
			// todo
			// 只有存的指针的时候进行清除
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
				
			// 从k地址清除若干个子节
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
			
			// 逻辑和k一样
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				// 非堆上内存
				memclrNoHeapPointers(e, t.elem.size)
			}
			
			// 当前元素置为空
			b.tophash[i] = emptyOne
			
			// 判断下一个元素是不是emptyRest状态
			// 如果不是，直接删除完成，结束了
			// 当前元素是当前桶最后一个，判断溢出桶的第一个
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
			
				// 判断当前桶的下一个
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			
			// 依次向前将寻找状态是emptyOne
			// 将状态改为emptyRest
			for {
				//	当前的元素肯定是最后一个了
				b.tophash[i] = emptyRest
				if i == 0 {
					// 如果不在溢出桶，且是最后一个
					// 说明是当前这个hash后缀的第一个
					// 第一个都变成emptyRest，不需要再向前了
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					
					// 拿到前面的桶
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					
					// 拿到前面的桶以后，从桶最后一个元素向前遍历
					i = bucketCnt - 1
				} else {
					i--
				}
				
				// 如果是emptyOne，
				// 因为它后面的元素已经是emptyRest，那它也可以变成emptyRest
				// 否则的话就break，因为这个元素不是emptyOne
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}

	// ？
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	
	// 修改状态，删除结束
	h.flags &^= hashWriting
}
// map的删除是标记删除，依旧占用空间，只有在扩容的时候才会真正的去挪动kv
// 没有看到map的缩容
```

```go
// map迭代
func mapiterinit(t *maptype, h *hmap, it *hiter) {

	// 初始化
	it.t = t
	it.h = h

	// grab snapshot of bucket state
	// 这个时候map可能正在扩容呢
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.ptrdata == 0 {
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
	// 迭代的起点
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	// 因为有可能存在另外一个迭代器，所以map的状态可能已经被改了。被改了就不用再改了
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}

// map迭代的时候是先获得一个hiter，然后不停调用mapiternext
// 每次调用后获取it的k和v，获取不到了就结束了
func mapiternext(it *hiter) {
	h := it.h
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket

next:
	// 迭代第一路 或者上个桶已经迭代完了
	if b == nil {
		//	绕了一圈又回来了
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.elem = nil
			return
		}
		
		// 正在扩容的时候进行迭代，这时候可能需要去旧的桶，也可能需要去新的桶
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			// 拿到旧的桶列表。因为上面定义的时候就可能在处于扩容中，所以bucket不可信
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			
			// 如果当前的b没有因为扩容被挪走。那就直接可以去旧桶拿了
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				// 已经被挪走了，b就等于新的桶
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			// 1、不在扩容
			// 2、扩容，但是发生在迭代之后进行了翻倍扩容
			// 两种情况都直接在新的桶里面取
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		// 桶号++。
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			// 再次回到起始桶，说名已经迭代完毕了
			it.wrapped = true
		}
		i = 0
	}
	for ; i < bucketCnt; i++ {
		// 根据随机偏移量计算每个桶起始位置
		// & 运算代替模运算
		offi := (i + it.offset) & (bucketCnt - 1)
		
		// 不管因为什么原因，这个节点是空的
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// TODO: emptyRest is hard to use here, as we start iterating
			// in the middle of a bucket. It's feasible, just tricky.
			continue
		}
		
		// 不是空的，就拿k和v
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey() {
			k = *((*unsafe.Pointer)(k))
		}
		e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
		
		// 上面判断了在扩容时新旧桶一样，且数据没挪走才会 checkBucket != noCheck
		// 扩容时新旧桶一样，则扩容是发生在迭代之前的。
		// 这里还判断的非等量扩容
		if checkBucket != noCheck && !h.sameSizeGrow() {
			// 如果没有math.NaN()类型，
			if t.reflexivekey() || t.key.equal(k, k) {
				// 重新计算以后的新桶和旧桶不同了
				hash := t.hasher(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
				// 这里要判断挪动以后k去了哪个桶，去了Y桶，就不能用了
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		
		// 当前的k没有被拿走
		// 或者当前的k是不可重现的
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			
			// t.key.equal(k, k) 只有在k=math.NaN()成立
			// b := math.NaN()
			// c := math.NaN()
			// b == c => false
			// 说明这个k不能进行查找
			// 这种情况计算出来的hash是不一样的
			// 这个地方就是判断这个值如果是math.NaN()类型，就直接可以取出
			// 因为math.NaN()不能通过mapaccessK找到
			
			// 这里要表达的意思就是即使当前的key已经改成了evacuatedX或者evacuatedY还要从这里拿
			// 因为这个key是不可以重现的，也就是hash以后的结果就不一样了。不能通过mapaccessK获取
			
			// 其他的key为什么从这里拿，即使状态evacuatedX或者evacuatedY，这个元素也是可访问的
			// 因为这个key可能已经变更或删除了，修改后不会更新扩容之前的。
			// 而这个不可重新的key刚好又不用担心被删除或者被更新了。
			!(t.reflexivekey() || t.key.equal(k, k)) {
			it.key = k
			if t.indirectelem() {
				e = *((*unsafe.Pointer)(e))
			}
			it.elem = e
		} else {
			// 当前的k被挪走了，要再查一次，因为现在时不知道怎么挪动的
			// 有可能发生的事情时挪动和迭代同时
			// 这个才挪走一半，迭代就开始了
			// 可以看到迭代其实是不能和写入同时进行的，迭代也被当作读
			// map要么就单线程操作，要么就只能一次初始化后续只能读
			rk, re := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.elem = re
		}
		
		// k， v 都已经拿到了，
		// 保留一些现场，用来下次迭代
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	
	// 当前桶已经迭代完了，去溢出桶继续迭代
	b = b.overflow(t)
	i = 0
	goto next
}
```
