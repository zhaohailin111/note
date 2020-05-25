# map
1.14.2

**拉链法**

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
