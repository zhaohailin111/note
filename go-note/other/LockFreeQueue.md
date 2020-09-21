# LockFreeQueue
基于https://github.com/smallnest/queue/blob/master/lockfree_queue.go 实现分析

```go
// Package queue provides a lock-free queue and two-Lock concurrent queue which use the algorithm proposed by Michael and Scott.
// https://doi.org/10.1145/248052.248106.
//
// see pseudocode at https://www.cs.rochester.edu/research/synchronization/pseudocode/queues.html
// It will be refactored after go generic is released.
package queue

import (
	"sync/atomic"
	"unsafe"
)

// LKQueue is a lock-free unbounded queue.
// 无锁队列，包含一个队列头和队列尾
type LKQueue struct {
	head unsafe.Pointer
	tail unsafe.Pointer
}

// 每个节点包含next指针和具体的value
type node struct {
	value interface{}
	next  unsafe.Pointer
}

// NewLKQueue returns an empty queue.
// 创建一个空的队列
func NewLKQueue() *LKQueue {
	n := unsafe.Pointer(&node{})
	return &LKQueue{head: n, tail: n}
}

// Enqueue puts the given value v at the tail of the queue.
// 向队列中放入数据
func (q *LKQueue) Enqueue(v interface{}) {
	n := &node{value: v}
	for {
		// 获取队列尾和下一个节点指针
		tail := load(&q.tail)
		next := load(&tail.next)
		// 这里再做一次判断是防止队列尾部发生变化
		if tail == load(&q.tail) { // are tail and next consistent?
			// next节点是空指针，说明当前获取的tail是真正的队尾部
			if next == nil {
				// cas将当前生成的节点替换这个next空节点。
				if cas(&tail.next, next, n) {
					// 替换成功以后，tail就不是最后一个节点了，需要向后移动
					// 这里是否会失败呢
					// 1、先假设会失败，因为不失败是没问题的
					// 如果会失败，那么尾部指针没有向后移动
					// 如果没有向后移动，那么q.tail就不是真正的tail
					// 如果不是真正的tail，那么下次获取的next就不是空
					// 如果不是空就会走下面的else逻辑
					// 下面的else同样会将tail向后移动。所以失败是没有任何问题的
					// 2、为什么会失败？
					// 失败就是因为当前的tail不是q.tail，说明在修改next的同时，有人修改了tail
					// 不过从入队列来看，修改tail只有两种情况
					// a、修改了next。必定只有当前的gr能修改next，这个不成立
					// b、next不为空。因为这次cas失败，才会next不为空，这个不成立
					// 入队不会产生这个异常，出队同样没有异常
					
					// 再整体来看修改队尾的情况
					// 1、next不为空，队尾后移动
					// 2、next后移，队尾后移
					// 3、出队时判断head==tail并且next不为空，队尾后移
					// 再看1，2，3同时发生的时候在2发生，next后移，此时有入队或者出队操作，发现next不为空，可以后移，触发后移，导致这个cas失败。
					
					// 失败原因找到了，那么1，2，3是否都是有必要
					// 1、去掉此处的cas会发生什么：此处没有后移，出队或者再次入队后移，导致后移延迟
					// 2、去掉下面else处的，会导致后移延迟
					// 3、去掉出队处的，会导致明明有数据，却不能出队
					cas(&q.tail, tail, n) // Enqueue is done.  try to swing tail to the inserted node
					return
				}
			} else { // tail was not pointing to the last node
				// try to swing Tail to the next node
				// 不要这步行不行？
				cas(&q.tail, tail, next)
			}
		}
	}
}

// Dequeue removes and returns the value at the head of the queue.
// It returns nil if the queue is empty.
// 出队，从对头出队
func (q *LKQueue) Dequeue() interface{} {
	for {
		head := load(&q.head)
		tail := load(&q.tail)
		next := load(&head.next)
		// 和入队一样
		if head == load(&q.head) { // are head, tail, and next consistent?
			// 队头和队尾相同，可能是空队列
			if head == tail { // is queue empty or tail falling behind?
				// next为空，说明真的没有数据了
				if next == nil { // is queue empty?
					return nil
				}
				// next不为空，说明队尾不是真正的队尾向后移动。
				// tail is falling behind.  try to advance it
				cas(&q.tail, tail, next)
			} else {
				// read value before CAS otherwise another dequeue might free the next node
				// 头尾不相同，说明next必然不为空？
				// 可以在入队看到，入队是先交换next再交换队尾，所以队尾已经不是队头了，那么next一定不为空
				// 直接交换对头和next指针
				v := next.value
				if cas(&q.head, head, next) {
					return v // Dequeue is done.  return
				}
			}
		}
	}
}

func load(p *unsafe.Pointer) (n *node) {
	return (*node)(atomic.LoadPointer(p))
}

func cas(p *unsafe.Pointer, old, new *node) (ok bool) {
	return atomic.CompareAndSwapPointer(
		p, unsafe.Pointer(old), unsafe.Pointer(new))
}
```
