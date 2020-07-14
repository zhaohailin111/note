#GC

## 个人理解

**保证三色不变形性来保护内存不会错误回收**

当一切按照正常轨迹走，可能发生的场景（黑色 -> 灰色 -> 白色）：

- 白色对象指向黑色对象
- 白色对象指向灰色对象
- 白色对象指向白色对象
- 灰色对象指向黑色对象
- 灰色对象指向灰色对象
- 灰色对象指向白色对象
- 黑色对象指向黑色对象
- 黑色对象指向灰色对象
- 黑色对象指向白色对象

有且只有```黑色对象指向白色对象```是不合法的

黑色对象不会重新扫描了，当黑色指向白色，白色被错误回收。

三色不变性

- 强三色不变性 — 黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象
- 弱三色不变性 — 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径

说白了就是打破`当黑色指向白色，白色被错误回收`规则：**当黑色对象指向白色对象时，这个白色对象一定还有一个灰色祖先，否则黑色对象不能指向白色对象**

golang 的策略：要满足弱三色不变性

> The weak tricolor invariant observes that it's okay for a black object to point to a white object, as long as some path ensures the garbage collector will get around to marking that white object.

- 插入写屏障：当出现黑->白，那就变成黑->灰。这样就需要每次指针操作都要把儿子标记为灰

- 删除写屏障：当出现黑->白，那就变成灰->白，这样就需要每次指针操作都要把父亲标记为灰

以上两种写屏障都是可以绝对保证三色不变性的。

### golang的混合写屏障

>shade(ptr) marks the object at ptr grey if it is not already grey or black.

#### 1.7的gc

引入插入写屏障，所有指针操作，都将目标节点变灰，满足强三色不变性。

如果是在所有内存操作都这样做，那肯定是不会有任何问题的，但是关键的golang只选择的在堆上这样做。

那就会出现下面情况：

- 堆上黑色节点指向任意白色节点，都会把白色节点变灰

- 栈上黑色节点指向任意白色节点，什么都不做。

那这样会出现什么问题呢：

- 无论是堆还是栈上白色，被堆上节点引用，都变成了灰色。这样这个白色断掉其他任意灰色祖先的边，是没问题的

- 无论是堆上还是栈的白色，被栈上黑色引用，不会发生改变，这样这个白色一旦失去了其他的灰色节点的父亲。最终会被错误回收，因为他是可以被栈引用的。

对于上述第二个问题，golang使用了重新扫描去解决。也就是gc最大的性能问题。

标记结束以后，重新扫描所有栈，把上述第二个问题导致的白色节点变成黑色。

#### 1.8的gc

golang的混合写屏障：

结合删除写屏障，golang这样做

```go
writePointer(slot, ptr):
    shade(*slot)
    shade(ptr)
    *slot = ptr
```

- 对于上面第二个问题，栈黑指向堆白，这个堆白一定是可以访问的，也就是一定有一个灰色祖先。那么当前就是一个栈黑指向堆白，一个堆灰指向堆白。此时如果这个堆灰指向的堆白断开，则可以描述为 堆灰:=new(obj)，此时将堆灰指向的堆白变成灰色，这样就是所谓的删除写屏障。因为是断开以后将儿子结点变成灰色，也就是伪代码中的shade(*slot)，这里的slot是\*slot也就是slot之前所指向的节点。omg。至此解决了栈黑->堆白的问题。

- 最后一个问题。栈黑->栈白。栈灰断开与栈白的连接。todo

改进：todo

```go
writePointer(slot, ptr):
    shade(*slot)
    if any stack is grey:
        shade(ptr)
    *slot = ptr
```

只有堆写屏障，没有栈写屏障？

这个问题也是没有得到一个合理的解释

>However, it also has disadvantages. In particular, it presents a trade-off for pointers on stacks: either writes to pointers on the stack must have write barriers, which is prohibitively expensive, or stacks must be permagrey.

> Thus, at the end of the cycle, the garbage collector must re-scan grey stacks to blacken them and finish marking any remaining heap pointers. 


todo？

两次stw：

- 第一次为什么必要？需要使新创建的节点变成黑，这个不能和用户程序同时进行。其他还有吗？

- 第二次为什么必要？用户程序和标记同时进行，应该会出现无休止的标记？不会的，因为新节点都是黑的，那可以得出所有节点终将变黑，而```shade(ptr) marks the object at ptr grey if it is not already grey or black.```。为什么呢？



[https://studygolang.com/articles/25205](https://studygolang.com/articles/25205)

[https://go.googlesource.com/proposal/+/master/design/17503-eliminate-rescan.md](https://go.googlesource.com/proposal/+/master/design/17503-eliminate-rescan.md)
