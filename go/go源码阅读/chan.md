# channel

## channel的数据结构
- channel的底层数据结构是hchan

```
    type hchan struct {
    	qcount   uint           // 环形队列中现有元素的个数
    	dataqsiz uint           // 环形队列能够容纳的元素个数
    	buf      unsafe.Pointer // 指向dataqsiz个数的数组中的一个元素
    	elemsize uint16         // 环形队列中每一个元素的大小
    	closed   uint32         // 标记channel是否关闭
    	elemtype *_type         // 元素的类型
    	sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
    	recvx    uint           // 队列下标，指示元素从队列的该位置读出
    	recvq    waitq          // 等待读消息的goroutine队列
    	sendq    waitq          // 等待写消息的goroutine队列
    
    	// lock保护hchan中的所有字段, 以及阻塞在这个channel中的sugos的一些字段
    	// 当持有lock时，不应该改变其他G的状态，(特别的，不能ready一个G)，因为它会在栈收缩时发生死锁
    	lock mutex // 互斥锁  不允许并发读写
    }
     type waitq struct {
        first *sudog
        last  *sudog
     }
```


## channel的创建
- 通过make创建的channel，创建channel的make函数底层是makechan，无论创建有缓存的channel还是无缓存的channel都是由makechan创建的，
而无缓冲的channel和有缓冲的channel的区别就是buf大小，无缓冲的channel的buf是空的
- 创建的hchan是以8字节方式内存对齐，如果创建的buf不包含指针，那么会一起创建hchan和buf，buf紧跟hchan后，并且元素是持久的，不会被gc回收。
````
/**
 * 创建通道
 * @param t 通道类型指针
 * @param size 通道大小，0表示无缓冲通道
 * @return
 **/
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// 编译器进行安全检查
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	// 如果不是以8字节方式内存对齐 或者elem的对齐方式大于8字节  会throw
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	// 计算channel需要分配的内存
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	// overflow表示堆栈溢出
	// mem > maxAlloc-hchanSize 表示分配的内存 超出了限制
	// size < 0 不允许size小于0
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// 当存储在buf中的元素不包含指针时，Hchan中不包含gc感兴趣的指针,buf指向相同的分配，元素是持久的
	// sudog 是从他们自己的线程中引用的 因此不能收集他们
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// 队列或元素大小为零。
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		//在此位置使用竞争检测器是为了同步
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// 元素不包含指针。 在一次调用中分配hchan和buf。
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize) // buf指针指向c+hchanSize的位置  buf是紧跟在hchan后的一段连续空间
	default:
		// 元素包含指针。
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)   // 设置元素大小
	c.elemtype = elem                //设置元素类型
	c.dataqsiz = uint(size)          //设置channel大小
	lockInit(&c.lock, lockRankHchan) // TODO 不懂是什么意思

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}

````

## 发送 chansend

- 当我们执行c <- x时，编译器会调用chansend1完成发送操作。chansend1调用chansend。
- 向一个空的channel发送数据，那么goroutine会通过gopark被挂起。
- send和recv方法中都有一个fastpath的校验方式，就是在加锁之前判断，如果当发送和接收时是非阻塞的情况，
并且没有可以发送和接收的数据，发送和接收的数据，那么那么会直接返回
- 发送数据分为四种情况
    - 如果hchan中有等待接收的接收者（这种情况是队列中的buf是空的），那么直接将接收者出队，将数据直接拷贝给接收者，并通过gorady唤醒接收者的goroutine
    - 如果hchan中的buf有足够的空间容纳发送的数据，那么直接将数据放入到buf中。
    - 如果hchan中的buf没有足够的空间容纳要发送的数据，如果是非阻塞的，直接返回，返送失败。
    - 如果是可以阻塞的，那么将发送的goroutine加入sendq，并通过gopark挂起。
- 向一个已经关闭的channel中发送数据，会panic。

````
    /*
     * 通用单通道发送/接收
     * 如果block不为nil，则protocol将不会休眠，但如果无法完成则返回。
     * 当涉及休眠的通道已经关闭的时候，可以使用g.param == nil唤醒休眠，最容易循环并重新运行该操作，我们将看到他现在已经关闭
     * @param c 通道对象
     * @param ep 元素指针
     * @param block 是否阻塞
     * @param callerpc 调用者指针
     * @return bool true：表示发送成功
     */
    func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
        // channel 已经空了
        if c == nil {
            // 非阻塞
            if !block {
                return false
            }
            // 将当前goroutine置于等待状态并调用unlockf。 如果unlockf返回false，则继续执行goroutine程序。
            // unlockf一定不能访问此G的堆栈，因为它可能在调用gopark和调用unlockf之间移动。
            // Reason参数说明了goroutine已停止的原因。
            // 它显示在堆栈跟踪和堆转储中。
            // Reason应具有唯一性和描述性。
            // 不要重复使用waitReason，请添加新的waitReason。
            gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
            throw("unreachable")
        }
        
        // Fast path: 在没有获取锁的情况下检查失败的非阻塞操作
        //
        // 在观察到channel尚未关闭，我们观察到channel还没有准备好send，这些观察中每一个都是单个字读取(第一个的c.closed和第二个的full())
        // 因为一个关闭的channel无法从'ready for sending'转换成'not ready for sending'，即使channel在两个观测值之间处于关闭状态，他们也隐含着一个时刻，
        // 即channel还没有关闭，但是还没有准备好发送。我们的行为就好像我们观察这个时刻的channel，并报告发送不能进行。
        // 此处对读取重新排序也是可以的，如果我们观察还没有准备好send并且还没有关闭，以为着channel在第一次观察结果中并没有关闭，然而，并没有任何东西保证取得进展
        // 我们依靠chanrecv()和closechan()中锁释放的副作用来更新c.closed and full()的线程视图
        if !block && c.closed == 0 && full(c) {
            return false
        }
    
        var t0 int64
        if blockprofilerate > 0 {
            t0 = cputicks()
        }
    
        // 加锁
        lock(&c.lock)
    
        // 加锁后 重新进行检查 如果channel已经关闭 那么 解锁 panic
        if c.closed != 0 {
            unlock(&c.lock)
            panic(plainError("send on closed channel"))
        }
    
        // 获取等待接收的队列
        if sg := c.recvq.dequeue(); sg != nil {
            // 找到了等待的接收者。 我们绕过通道缓冲区（如果有）将要发送的值直接发送给接收器。
            send(c, sg, ep, func() { unlock(&c.lock) }, 3)
            return true
        }
    
        // 检查队列中是否有空间容纳新的元素
        if c.qcount < c.dataqsiz {
            // Space is available in the channel buffer. Enqueue the element to send.
            // 根据sendx找到要存储的数据在环形队列buf中的位置
            qp := chanbuf(c, c.sendx)
    
            // 拷贝数据到指定位置
            typedmemmove(c.elemtype, qp, ep)
            // 队列下标++ 指向下一个要存储的位置
            c.sendx++
    
            // 如果下表已经达到队列长度的位置 需要将sendx重新移动到队列头   越界判断
            if c.sendx == c.dataqsiz {
                c.sendx = 0
            }
    
            // 队列内已经存储的数目++
            c.qcount++
            unlock(&c.lock)
            return true
        }
        // 走到这里 就证明 没有等待接收的 而且环形队列中没有空间容纳新的元素  那么需要将当前的发送队列阻塞
        // 如果有非阻塞的标志  那么直接解锁 返回发送失败
        if !block {
            unlock(&c.lock)
            return false
        }
    
        // Block on the channel. Some receiver will complete our operation for us.
        // 阻塞在channel中，一些接受者将为我们完成我们的操作  这里的意思是，当出现接收者时，会唤醒当前的goroutine
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
    
        // 加入到发送队列中
        c.sendq.enqueue(mysg)
        // gopark  休眠goroutine
        gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
        // Ensure the value being sent is kept alive until the
        // receiver copies it out. The sudog has a pointer to the
        // stack object, but sudogs aren't considered as roots of the
        // stack tracer.
        //确保发送的值保持活动状态，直到接收者将其复制出来。
        //sudog具有指向堆栈对象的指针，但是sudog不被视为堆栈跟踪器的根。
        KeepAlive(ep)
    
        // someone woke us up.
        if mysg != gp.waiting {
            throw("G waiting list is corrupted")
        }
        gp.waiting = nil
        gp.activeStackChans = false
        if gp.param == nil {
            if c.closed == 0 {
                throw("chansend: spurious wakeup")
            }
            panic(plainError("send on closed channel"))
        }
        gp.param = nil
        if mysg.releasetime > 0 {
            blockevent(mysg.releasetime-t0, 2)
        }
        mysg.c = nil
        releaseSudog(mysg)
        return true
    }

    // send在一个空的channel c中完成send操作
    // 将发送方发送的ep拷贝到接收方sg中
    // 然后将接收器唤醒 继续前进
    // channel c必须是空的 并且被lock  send通过unlockf参数解锁channel
    // sg 必须已经在channel c中出队
    // ep必须是非空的  并且指针指向堆 或者调用者的堆栈
    func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
        // 元素不为空  直接发送
        if sg.elem != nil {
            sendDirect(c.elemtype, sg, ep)
            sg.elem = nil
        }
        gp := sg.g
        unlockf()
        gp.param = unsafe.Pointer(sg)
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        /// 复始一个 goroutine，放入调度队列等待被后续调度
        // 第二个参数用于 trace 追踪 ip 寄存器的位置，go runtime 又不希望暴露太多内部的调用，因此记录需要跳过多少 ip
        goready(gp, skip+1)
    }

    // 在一个无缓冲通道或空缓冲通道发送和接收是一个正在运行的goroutine写入另一个正在运行的goroutine的栈上的唯一操作
    //
    func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
        // src is on our stack, dst is a slot on another stack.
    
        // Once we read sg.elem out of sg, it will no longer
        // be updated if the destination's stack gets copied (shrunk).
        // So make sure that no preemption points can happen between read & use.
        // 我们必须在一个函数调用中完成 sg.elem 指针的读取，否则当发生栈伸缩时，指针可能失效（被移动了）。
        dst := sg.elem
    
        // 为了确保发送到数据能够立刻被观察到 需要写屏障的支持 执行写屏障  保证代码的正确性
        typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
        // No need for cgo write barrier checks because dst is always
        // Go memory.
        // 写入receiver的栈中
        memmove(dst, src, t.size)
    }
````
## 接收 chanrecv

- <-c会被编译器编译成chanrecv1和chanrecv2，都是调用的chanrecv
- <-c如果是v，ok<-c这种使用方式，那么会编译成chanrecv2，chanrecv有两个返回值，第一个返回值代表是否被select选中，第二个返回值代表是否接收到数据
- chanrecv如果没有接收者，那么接收掉的数据会被舍弃
- 如果有接收者，如果channel已经关闭，那么接收者的值为0，否则接收者的值为接收到的数据的值
- 如果接收一个nil的channel，那么goroutine会被永久的阻塞
- chanrecv也在阻塞前进行fastpath的判断，如果非阻塞，channel没有关闭，并且没有能够接收的数据，那么返回的值为false，false
如果channel已经关闭，会对接收者内存清理，并返回true，false，代表被select选中，但是没有接收到数据
- chanrecv有五种情况
    - hchan中的等待发送的队列不为空，将发送者出队，如果hchan是无缓冲的，那么直接将要发送的数据拷贝给接收者，并通过goready唤醒阻塞的发送goroutine
    - 如果是有缓冲的，那么将队头的数据拷贝给接收者，将阻塞的发送goroutine的数据加入到队尾，并通过goready唤醒。
    - 如果队列中没有阻塞的发送者，并且buf队列不为空，那么直接将队头的数据拷贝给接收者
    - 如果buf队列是空的，接收者goroutine将要通过gopark挂起，如果是非阻塞的那么直接返回false，false
    - 如果是阻塞的，那么将接收者goroutine入接收者队列，然后将接收者goroutine挂起。等待有数据发送时唤醒。
    
````
// chanrecv 在通道c上接收 并将接收到的数据写入
// ep可能是空的 这种情况下 接收到的数据会舍弃掉
// 如果是非阻塞的  并没有没有可用的元素 返回false false
// 否则 如果c关闭  *ep = 0 并返回 (true, false).
// 否则 *ep被填充一个数据并返回
// 一个非空的ep必须在堆上 或者在调用者的栈上
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}
	// 如果接收一个空的channel  那么会永远的阻塞
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
	// fastpath  不需要加锁 就可以检查操作非阻塞操作
	// Fast path: check for failed non-blocking operation without acquiring the lock.
	if !block && empty(c) {
		// After observing that the channel is not ready for receiving, we observe whether the
		// channel is closed.
		//观察到channel还没有为接收者准备好时（也就是channel中既没有等待发送的 环形队列中也没有数据），观察channel是否已经关闭掉了
		// 因为一个关闭的通道 不可能再次打开 所以当通道关闭时 直接返回 false false
		// 如果通道没有关闭  但是没有为接收准备好  如果ep不是空的 那么需要清空ep  这个条件 只有在select 语句中才能发生并且select语句中要有default
		// Reordering of these checks could lead to incorrect behavior when racing with a close.
		// For example, if the channel was open and not empty, was closed, and then drained,
		// reordered reads could incorrectly indicate "open and empty". To prevent reordering,
		// we use atomic loads for both checks, and rely on emptying and closing to happen in
		// separate critical sections under the same lock.  This assumption fails when closing
		// an unbuffered channel with a blocked send, but that is an error condition anyway.
		if atomic.Load(&c.closed) == 0 {
			// Because a channel cannot be reopened, the later observation of the channel
			// being not closed implies that it was also not closed at the moment of the
			// first observation. We behave as if we observed the channel at that moment
			// and report that the receive cannot proceed.
			return
		}
		// The channel is irreversibly closed. Re-check whether the channel has any pending data
		// to receive, which could have arrived between the empty and closed checks above.
		// Sequential consistency is also required here, when racing with such a send.
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)
	// 上锁之后 还需要再一次检查channel是否已经关闭了 如果channel已经关闭了 并且环形队列是空的  如果ep不为空 那么清空ep  返回true false
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

	if sg := c.sendq.dequeue(); sg != nil {
		// 当发现有发送者  并且是无缓冲的 那么直接从发送这接收数据
		// 否则 从队列头中获取数据  并且将一个send的值 塞入到队列的尾部
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	//如果没有阻塞的发送者 并且环形队列中有数据 那么直接从队列中接收数据
	if c.qcount > 0 {
		// Receive directly from queue
		// 定位到接收的位置
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}

		// 如果ep不为空 那么直接拷贝到ep中
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}

		// 清空qp
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	// 如果上锁之后 队列中没有数据了 并且是非阻塞的 那么直接返回false false
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
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
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
// 执行recv的  channel 必须是full状态并且上锁的状态  通过unlockf参数解锁 sg已经从发送队列中出对了  非空的ep  必须在堆上 或者在调用者的栈中
// recv 分为两种情况
// 1。 recv 在无缓冲队列中  直接从发送者接收数据  唤醒发送者
//2。 recv从从环形队列中接收数据， 然后将发送者的数据写入到环形队列中  唤醒发送者
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
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// dst is on our stack or the heap, src is on another stack.
	// The channel is locked, so src will not move during this
	// operation.
	src := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}
````
## 关闭channel

- 关闭一个空的channel会panic
- 关闭一个已经关闭的channel会panic
- 关闭的步骤，释放所有的接收者队列，释放所有的发送队列，此时还没有真正的释放阻塞的goroutine，释放所有的goroutine，这时会将所有的锁住的channel解锁。
但是所有的发送者的goroutine会被panic。
````
func closechan(c *hchan) {
// 1。关闭一个空的channel 会panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	// 2关闭一个已经关闭的channel 会panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	c.closed = 1

	var glist gList

	// 释放所有读队列
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
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// 释放所有写队列 所有向channel中写的goroutine  会发生panic
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
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	// 释放所有的g  就可以释放所有的channel锁
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}

````
## 非阻塞的send和recv操作

- 当channel的接收和发送操作在一个拥有default的select中，c <- v会被编译器编译成selectnbsend，v=<-c会被编译成selectnbrecv，v，ok=<-c会被编译成selectnbrecv2，
底层用的都是chansend和chanrecv

````
// compiler implements
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}

// compiler implements
//
//	select {
//	case v = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbrecv(&v, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

// compiler implements
//
//	select {
//	case v, ok = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if c != nil && selectnbrecv2(&v, &ok, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
	// TODO(khr): just return 2 values from this function, now that it is in Go.
	selected, *received = chanrecv(c, elem, false)
	return
}
````










