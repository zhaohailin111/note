# map
1.14.2

**拉链法**

**对于hash以后的结果，如果后缀相同的，放在一个桶里面，也就是拉出一个链表，桶的大小是8。如果拉出的链表大于8会生成一个溢出桶。溢出桶会决定什么时候扩容**

**扩容触发依赖总数量和溢出桶数量**

- 总数量过多会触发扩容
- 溢出桶数量过多会触发扩容（防止出现总数量不多，但是hash算法不良导致都跑到一个桶里面）

```go
// A header for a Go map.
type hmap struct {
	count     int // map中元素数量
	flags     uint8
	B         uint8  // map中桶数量
	noverflow uint16 
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // 存储数据的桶s
	oldbuckets unsafe.Pointer // 用来扩容的桶s
	nevacuate  uintptr        

	extra *mapextra // optional fields
}
```
```go

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
```

```go
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
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

```go
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
		
		// 旧桶，如果旧的桶中还有元素，则在旧的桶里面寻找？
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
// 写入，当map[k]在左边的时候，返回的地址然后赋值
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
```
