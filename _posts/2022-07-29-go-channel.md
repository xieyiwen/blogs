---
layout: post
title: Go基本类型-Channel
categories: ["Golang","Channel","管道"]
date: 2022-07-29
keywords: ["golang", "channel"]
---

通过源码分析Go的基本类型-Channel（管道）,源代码版本：go 1.17.3.

## 一、管道介绍
Go中常见的原则是：不要通过共享内存进行通信，而是通过通信来共享内存。

Go实现了一种通讯顺序进程（Communicating sequential processes, CSP）,Goroutine和channel分别对应CSP中的实体和管道，Goroutine通过channel传递消息。

<img src="/assets/images/20220729/csp.png" width="60%">

G1向channel发送消息，G2从channel获取G1的消息，其中G1和G2相互独立运行，但是通过channel实现消息传递。

在Go中，channel收发操作遵循FIFO（先进先出）：
1. 先向channel读取数据的goroutine先收到数据；
2. 先向channel发送数据的goroutine先写入数据。


## 二、数据结构
channel是由：mutex、环状数据缓存、一个发送队列和一个接收队列组成，结构体为：

```text
// src/runtime/chan.go
type hchan struct {
	qcount   uint           // channel中的元素个数
	dataqsiz uint           // 循环队列的长度
	buf      unsafe.Pointer // 循环队列的指针
	elemsize uint16         // 元素的大小
	closed   uint32         // 关闭标志
	elemtype *_type         // 元素类型
	sendx    uint           // 发送操作处理到的数据的下标
	recvx    uint           // 接收操作处理到的数据的下标
	recvq    waitq          // 等待接收的goroutine列表
	sendq    waitq          // 等待发送的goroutine列表
	lock mutex
}
```

<img src="/assets/images/20220729/channel_struct.png" width="60%">

等待接收和发送数据的waitq结构定义：
```text
type waitq struct {
	first *sudog
	last  *sudog
}
```
可以看到，waitq定义了前后指针，从而使等待的goroutine构成了链表。其中队列中的goroutine信息封装在`runtime.sudog`中。

## 三、创建管道
Go的编译器将完成如下翻译：
`make(chan type, n) => makechan(type, n)`

Go中通过关键字`make`来初始化管道。其中编译器会将其转换成OMAKE指令，并在类型检查阶段将OMAKE转换成OMAKECHAN：
```text
// src/cmd/compile/internal/typecheck/typecheck.go
func typecheck1(n ir.Node, top int) ir.Node {
	if n, ok := n.(*ir.Name); ok {
		typecheckdef(n)
	}
	switch n.Op() {
	default:
		ir.Dump("typecheck", n)
		base.Fatalf("typecheck %v", n.Op())
		panic("unreachable")

	...

	case ir.OMAKE:
		n := n.(*ir.CallExpr)
		return tcMake(n)

	...
}
```

其中tcMake的定义为：
```text
// src/cmd/compile/internal/typecheck/func.go
func tcMake(n *ir.CallExpr) ir.Node {
	...
	i := 1
	var nn ir.Node
	switch t.Kind() {
	...
	case types.TSLICE:
		...
		nn = ir.NewMakeExpr(n.Pos(), ir.OMAKESLICE, l, r)
	case types.TMAP:
		...
		nn = ir.NewMakeExpr(n.Pos(), ir.OMAKEMAP, l, nil)
		nn.SetEsc(n.Esc())
	case types.TCHAN:
		l = nil
		if i < len(args) {
			// 携带缓冲区的设定，形如：make(chan int, 10)
			l = args[i]
			i++
			l = Expr(l)
			l = DefaultLit(l, types.Types[types.TINT])
			if l.Type() == nil {
			    // 形如: make(chan struct, 10)
				n.SetType(nil)
				return n
			}
			if !checkmake(t, "buffer", &l) {
				n.SetType(nil)
				return n
			}
		} else {
			// 形如: make(chan int)
			l = ir.NewInt(0)
		}
		nn = ir.NewMakeExpr(n.Pos(), ir.OMAKECHAN, l, nil)
	}
    ...
}
```

从tcMake可以看出，当在程序中使用make自定来初始化：slice、map、channel时，编译器都会将其分别转换为：OMAKESLICE、OMAKEMAP、OMAKECHAN分别处理。

在channel初始化中，会检查是否需要设置缓冲区，如果没有指定缓冲区，则默认设定为0表示channel无缓冲。

OMAKAECHAN在编译SSA中间代码阶段之前被转换成`runtime.makechan`,
```text
// src/cmd/compile/internal/walk/expr.go
func walkExpr1(n ir.Node, init *ir.Nodes) ir.Node {
	switch n.Op() {
	...
	case ir.OMAKECHAN:
		n := n.(*ir.MakeExpr)
		return walkMakeChan(n, init)
	...
}
// src/cmd/compile/internal/walk/builtin.go
func walkMakeChan(n *ir.MakeExpr, init *ir.Nodes) ir.Node {
	size := n.Len
	fnname := "makechan64"
	argtype := types.Types[types.TINT64]

	if size.Type().IsKind(types.TIDEAL) || size.Type().Size() <= types.Types[types.TUINT].Size() {
		fnname = "makechan"
		argtype = types.Types[types.TINT]
	}
	return mkcall1(chanfn(fnname, 1, n.Type()), n.Type(), init, reflectdata.TypePtr(n.Type()), typecheck.Conv(size, argtype))
}
```

如果缓冲区的size超过int32的取值范围，则使用`runtime.makechan64`。
```text
// src/runtime/chan.go
func makechan64(t *chantype, size int64) *hchan {
	if int64(int(size)) != size {
		panic(plainError("makechan: size out of range"))
	}
	return makechan(t, int(size))
}
```
可以看到，Go实际不支持int64的大小，而是强转为int。接下来看int大小的实现。

```text
// src/runtime/chan.go

const hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&7)

func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

    // 检查确认 channel 的容量不会溢出
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0:
		// 队列或元素大小为零
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// 元素不包含指针
		// 在一个调用中分配 hchan 和 buf
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 元素包含指针
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)
    ...
	return c
}

```
上述代码首先会检查需要申请的内存是否溢出（大于或等于2^32）,然后初始化hchan和缓冲区。
1. 如果无缓冲区，则只分别用于存储hchan的其他字段的内存，是最小的内存分配。
2. 如果管道中传递的类型不是指针类型，会为当前的 Channel和具体的数据类型分配一块连续的内存空间；
3. 其他情况，按照hchan和缓冲区大小分配内存；

而具体的 makechan 实现的本质是根据需要创建的元素大小，使用mallocgc分配内存， 因此Channel总是在堆上进行分配的，它们会被垃圾回收器进行回收，这也是为什么 Channel 不一定总是需要调用 close(ch) 进行显式地关闭。

## 四、发送数据
Go的编译器将完成如下翻译：
`ch <- v => chansend1(ch, v)`

编译器会将 `ch <- v`转换成OSEND指令，并在类型检查阶段将OSEND转换成`runtime.chansend1`：

```text
// src/cmd/compile/internal/walk/expr.go
func walkexpr1(n *Node, init *Nodes) *Node {
	switch n.Op {
	case ir.OSEND:
		n := n.(*ir.SendStmt)
		return walkSend(n, init)
	}
}
```
其中walkSend为：
```text
// src/cmd/compile/internal/walk/expr.go
func walkSend(n *ir.SendStmt, init *ir.Nodes) ir.Node {
	n1 := n.Value
	n1 = typecheck.AssignConv(n1, n.Chan.Type().Elem(), "chan send")
	n1 = walkExpr(n1, init)
	n1 = typecheck.NodAddr(n1)
	return mkcall1(chanfn("chansend1", 2, n.Chan.Type()), nil, init, n.Chan, n1)
}
// src/runtime/chan.go
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}
```
可以看到，`runtime.chansend1`单纯的调用了`runtime.chansend`：
```text
// @param c: channel
// @param ep: 发送的数据指针
// @param block: 是否阻塞
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	...
	// 快速路径：非阻塞操作下，channel未关闭且缓冲区已满，则快速返回失败
	if !block && c.closed == 0 && full(c) {
		return false
	}
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}
	lock(&c.lock)
    // channel已关闭
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
```

发送数据前需要竞争锁，然后判断channel是否已经被关闭，所以在工程上不能向closed的channel写数据。

接下来按照三种场景来看channel的发送数据操作：
1. 存在等待接收数据的goroutine，直接发送；
2. 缓冲区存在空闲，将数据写入到channel的缓冲区；
3. 不存在缓存区或缓冲区已满，阻塞。

上述三种场景是顺序执行的，首先来看第一种：
```text
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
```
如果存在receiver，则直接调用`runtime.send`来将数据传递给receiver。
```text
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	...
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```
发送时，需要完成两个操作：
1. 调用sendDirect， 将需要发送的数据拷贝给接收者。
2. 调用goready，设置接收者的状态为`_Grunnable`， 并作为下一个立即被执行的 Goroutine.

```text
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src来自发送者gp的栈，数据写入接收者的栈.
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}
```
接收方可能处于阻塞在channel中，所以发送数据后需要执行goready时调度器继续开始调度接收者。

然后看第二种：将数据写入缓冲区
```text
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
```
首先根据缓冲区中发送操作的当前坐标计算出数据在缓冲区中的内存地址，然后将需要发送的数据拷贝到缓冲区对应位置上中。然后sendx索引后移1为，qcount计算加1。

如果放入的位置是缓存区的最后一位，则sendx设置为0，结合qcount可以判断缓冲区是否已满。执行完成后，释放锁。

第三种情况：无缓冲区或缓冲区已满导致的阻塞：
```text
    // 如果不是阻塞发送，则直接释放锁，写入的数据丢弃。
	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
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
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	// 将当前的 g 从调度队列移出
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

    // 因为调度器在停止当前 g 的时候会记录运行现场，当恢复阻塞的发送操作时候，会从此处继续开始执行
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// 取消与之前阻塞的 channel 的关联
	mysg.c = nil
	// 从 sudog 中移除
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```
上述代码看起来比较复杂，整理如下：
1. 调用`runtime.getg`, 获取发送数据使用的goroutine。
2. 构建sudog，设置这次需要发送的相关信息。如果发送的channel、发送的goroutine、发送的数据、是否是select中等。
3. 将sudog加入发送等待队列，并设置到发送的Goroutine为sudog已就绪，等待发送；
4. 调用 runtime.gopark 将当前的 Goroutine 陷入沉睡等待唤醒；
5. 被调度器唤醒后会执行一些收尾工作，将一些属性置零并且释放 `runtime.sudog` 结构体。

```text
func chanparkcommit(gp *g, chanLock unsafe.Pointer) bool {
	// 具有未解锁的指向 gp 栈的 sudog
	// 栈的复制必须锁住那些 sudog 的 channel
	gp.activeStackChans = true
	unlock((*mutex)(chanLock))
	return true
}
```

函数在最后会返回 true 表示这次我们已经成功向 Channel 发送了数据。

总结下来，发送过程一共包含三步：
1. 获取channel的锁；
2. 发送者入缓冲队列，拷贝要发送的数据；
3. 释放锁。

其中第二部，包括：
1. 找到是否有正在阻塞的接收方，是则直接发送;
2. 找到是否有空余的缓存，是则存入;
3. 阻塞直到被唤醒。

## 五、接收数据
Go支持两种方式接收channel的数据。
1. `v <- ch`
2. `v, beforeClosed <- ch`

两种方式最终会编译为调用`runtime.chanrecv`。
```text
// src/runtime/chan.go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if c == nil {
	    // 如果channel未初始化，程序崩溃
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// 快速路径：在不需要锁的情况下检查失败的非阻塞操作
	if !block && empty(c) {
	    // 如果channel未关闭，但是channel为空，返回false
		if atomic.Load(&c.closed) == 0 {
			return
		}
		if empty(c) {
			// 竞态检查
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			// 如果channel已关闭，返回类型数据的默认值
			// received设置为false，表示数据不是来自发送者
			if ep != nil {
				// 清除ep的数据，相当于恢复默认值
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}
```
接收数据首先检查channel的情况，如果channel没有数组且是非阻塞接收，则可按照不同情况快速返回：
1. channel未初始化，会将当前的 Goroutine 休眠，从而发生死锁崩溃
2. channel未关闭，但是buf是空的，相当于channel没有数据，则直接返回false
3. channel已关闭，buf为空，返回默认值和false。

接着往下：
```text
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

    // 获取锁
	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
	    // 如果channel已关闭，且channel为空，释放锁，快速返回
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}
```
channel的接收是异步的，所以在快速判断之后，还需要在获得锁后再次判断是否为空。

接着往下：
```text
	if sg := c.sendq.dequeue(); sg != nil {
		// 如果有发送者，则直接从发送者出读取数据，然后返回
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
```
如果有发送者阻塞在channel中（上文发送数据的第三种情况），则直接从channel的发送者处接收数据，然后快速返回。

上文发送数据阻塞恢复后，可以直接清空sudog的设置，就是因为receiver已经把数据拿走了。

跳到`runtime.recv`：
```text
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
	    // 如果没有缓冲区
		if raceenabled {
			racesync(c, sg)
		}
		// 直接把发送者的数据复制给ep（栈->栈）
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// 直接从队列中接收
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
			racenotify(c, c.recvx, sg)
		}
		// 从队列中拷贝数据给接收者
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 缓存空出来一个位置了，把发送者的数据存入缓存
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	
	// 释放一个阻塞的发送者
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```
如果channel存在缓冲区，则接收者获取数据后，必然可以释放阻塞在channel中头部的发送者。

接着往下：
```text
	if c.qcount > 0 {
		// buf不为空
		// 从buf中获取接收数据
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
		    // 将数据复制给eq
			typedmemmove(c.elemtype, ep, qp)
		}
		// 清空已被接收的buf位置
		// 接收index后移1位
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// buf减1
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
```
类似发送数据，这是第二种情况，接收者可以从缓冲区中获取到数据，从而快速返回。

接着往下：
```text
    // 当这一步就会开始进行阻塞
    // 所以需要阻塞前判断是否需要继续，如果是非阻塞的receive，则直接返回false
	if !block {
		unlock(&c.lock)
		return false, false
	}
        
	// 没有可以使用的sender，开始阻塞
	gp := getg()
	// 构建sodug绑定到channel
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	// sudog放入channel的等待接收者队列
	c.recvq.enqueue(mysg)
	atomic.Store8(&gp.parkingOnChan, 1)
	// 将当前的 g 从调度队列移出
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// 唤醒，并清除配置
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```
和发送数据一样， 如果没有发送者，接收者会被阻塞到channel的缓冲区中，直到有发送者调用`runtime.chansend`。


## 六、关闭管道
Go支持显性调用close来关闭管道:`close(ch) => closechan(ch)`。

我们直接定位到closechan:
```text
// src/runtime/chan.go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	// 不能重复关闭
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}
    // 设置关闭标志
	c.closed = 1

	var glist gList

	// 释放阻塞的所有接收者
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
		    // 因为已经关闭了， 所以数据被擦除
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// 释放所有的发送者
	// 被释放的发送者会panic
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
		gp.param = unsafe.Pointer(sg)
		// 设置为失败
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// 释放锁以后再执行发送者和接收者的唤醒工作
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

跳出来思考两个问题。

**第一个问题**：channel是支持指针的，也就是说channel传递指针是拷贝的是指针地址，发送者和接收者拿到的指针指向的还是同一片内存地址，所以channel传递的指针类型并发读写是不安全的。

**第二个问题**：chansend和chanrecv都使用了非阻塞情况下的快速失败路径，为什么chanrecv需要atomic原子操作，而chansend不需要？

如果管道没有关闭，chansend失败的原因是：没有接收者或者缓冲区已满。chanrecv失败的原因是：没有发送者或缓冲区为空。

chanrecv的atomic保证了channel未关闭和缓冲区为空的顺序。 
首先chansend失败的结论是可以保证一致的，因为：channel关闭的情况下，chansend也会得到不能发送数据的结论。

但是chanrecv不可以，因为在channel被关闭的情况下，如果先判断缓冲区为空，则可能得到结论是：channel未关闭且缓冲区空。

如果先判断close被关闭了，那么得到的结论是：channel已关闭且缓冲区为空。这两者结论不一致，所以需要通过atomic来保证两个指令的编排顺序。

## 七、总结
通过对channel的解读，可以知道：
1. 不能往关闭的channel中发送数据;
2. 关闭的channel可以反复的读取数据，如果数据不是发送者发送的，则会得到received=false的标志;
3. 通过channel通信的数据，是从发送端的goroutine的栈中直接复制到接收端的goroutine的栈中的；
4. 存在非阻塞的channel使用，就是结合select；


## 八、参考

1. [Go 语言 Channel 实现原理精要 | Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)
2. [3.6 通信原语 | Go 语言原本](https://golang.design/under-the-hood/zh-cn/part1basic/ch03lang/chan/)
