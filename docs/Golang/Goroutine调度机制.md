# Goroutine调度机制
### GPM概念
GRQ：
global run queue 全局运行队列
LRQ：
local run queue 本地运行队列



### 基础概念，GPM执行过程
#### GM模型
![](http://img.codwiki.cn/20210428/640.gif)

<center>GM模型</center>

在GM模型中，内核线程M需要从全局队列中获取G，需要通过加锁的方式来解决并发安全的问题。因此在高并发下，锁会成为该模型的性能瓶颈。

<br desc/>

#### GPM模型

GPM模型在原来的GM模型的基础之上增加了一个中间层，当M想要获取可执行的G的时候，会优先从P(本地协程队列)中获取，因为P与M之间是绑定关系，所以获取过程无需加锁。

全局协程队列依然存在，当M无可执行的G时，会优先从全局队列（前提是全局队列不为空）中获取可执行的G，数量计算规则如下公式：
$$
min(\frac{len(全局队列)}{GOMAXPROCS + 1}, \frac{len(全局队列)}{2})
$$
当全局队列为空，M又无可执行的G时，会触发 <code>work stealing</code> 机制，从其他P的本地队列中偷取一半的G到自己绑定的P的本地队列中。



![GMP模型](https://img.codwiki.cn/20210428/GMP%E6%A8%A1%E5%9E%8B.gif)

<center>GMP模型</center>

>注意：P与M之间是绑定关系，但P与M的数量没有绝对的对应关系，当一个M阻塞的时候，P就会去创建或者切换到另外一个的M。

<br desc/>

#### hand off机制

当M线程因为G运行而阻塞时，M线程将释放绑定的P，P将创建或切换到其他空闲的M并与其绑定。



### 程序启动过程

程序入口：runtime/asm_amd64.s

```assembly
// runtime·rt0_go(SB)
```



### 创建goroutine

新创建的goroutine将会被放入本地队列的runnext里，原本在runnext里的g会被放入LRQ的尾部，如果队列已满，则放入全局队列。

runtime/proc.go

```go
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}

	if next {
    // 将创建的gp放入_p_.runnext里，会被优先执行
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
      // 如果有其他线程正在操作runnext将会重试
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
    // 将旧的runnext成员踢出，放入本地队列
		gp = oldnext.ptr()
	}

retry:
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
  // 判断本地队列是否已满
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
  // 本地队列已满，放入全局队列
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}
```

如果本地队列已满，会将本地队列的一半连同当前的gp一起放入全局队列。



```go
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
  // _p_本地队列的一半+gp
	var batch [len(_p_.runq)/2 + 1]*g

	// First, grab a batch from local queue.
	n := t - h
	n = n / 2
	if n != uint32(len(_p_.runq)/2) {
		throw("runqputslow: queue is not full")
	}
	for i := uint32(0); i < n; i++ {
		batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))].ptr()
	}
	if !atomic.CasRel(&_p_.runqhead, h, h+n) { // cas-release, commits consume
		return false
	}
	batch[n] = gp

	if randomizeScheduler {
		for i := uint32(1); i <= n; i++ {
			j := fastrandn(i + 1)
			batch[i], batch[j] = batch[j], batch[i]
		}
	}

	// Link the goroutines.
  // 将需要放入全局队列的g链接起来，减少锁时间
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}
	var q gQueue
	q.head.set(batch[0])
	q.tail.set(batch[n])

	// Now put the batch on global queue.
	lock(&sched.lock)
	globrunqputbatch(&q, int32(n+1))
	unlock(&sched.lock)
	return true
}
```

全局队列是一个链表，将放入全局队列的goroutine通过链表链接起来，然后与全局队列的队尾链接起来。



### 调度机制

当系统创建M的时候就会有g0，这个g0用的是system stack，而不是G的user stack.
g0执行schedule函数，用来从GRQ或者LRQ上寻找可执行的G
一个M执行G的过程与系统函数调用具有相似的过程

![image-20210429172950617](http://img.codwiki.cn/20210429/image-20210429172950617.png)

<center>每个M线程都将创建一个g0</center>



当通道阻塞时，当前正在执行的goroutine将被停止执行（处于等待状态），然后g0将会替换goroutine进行调度操作。

``` go
ch := make(chan int)
[...]
ch <- v
```



![image-20210429183659082](http://img.codwiki.cn/20210429/image-20210429183659082.png)



当处于等待状态的goroutine读取到消息之后，将立即解除等待状态。M线程将切换到g0，将驻留的goroutine放入本地队列当中。

``` go
v := <-ch
```



![image-20210429183852775](http://img.codwiki.cn/20210429/image-20210429183852775.png)

从go1.5开始，从阻塞通道返回的goroutine将优先被执行。



#### g0的职责：

1. 创建新的goroutine，并将其放入本地队列
2. defer方法分配
3. GC操作
4. 栈增长，增加goroutine栈的大小



当发生系统调用的时候，Go会将正在运行的线程进入阻塞模式，然后让新线程来处理当前P的本地队列。



#### 循环调度

``` go
func schedule() {
  _g_ := getg()
  ...
top:
  // gc等待
  if sched.gcwaiting != 0 {
    gcstopm()
    goto top
  }
  
  ...
  //全局队列有可能会出现长时间不被调度而饿死的情况，因此go通过调度计数器来每隔61次调度一遍全局队列。
  if gp == nil {
    if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
      lock(&sched.lock)
      gp = globrunqget(_g_.m.p.ptr(), 1)
      unlock(&sched.lock)
    }
  }
  
  // 从p的本地队列中获取等待执行的g
  if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
		if gp != nil && _g_.m.spinning {
			throw("schedule: spinning with local work")
		}
	}
  // 如果本地队列无可执行的g，尝试从其他地方获取g
	if gp == nil {
		gp, inheritTime = findrunnable() // blocks until work is available
	}
  ...
  // 执行
  execute(gp, inheritTime)
}
```



调度过程中有可能会出现一个问题，正在执行的goroutine占用线程时间过长，会造成其他的goroutine饥饿。



### 协作式调度

协作式调度依靠被调度方主动弃权，通过Gosched()方法来实现。

```go
// Gosched 会让出当前的 P，并允许其他 Goroutine 运行。
// 它不会推迟当前的 Goroutine，因此执行会被自动恢复
func Gosched() {
	checkTimeouts()
	mcall(gosched_m)
}
// Gosched 在 g0 上继续执行
func gosched_m(gp *g) {
	...
	goschedImpl(gp)
}

func goschedImpl(gp *g) {
	// 放弃当前 g 的运行状态
	status := readgstatus(gp)
	...
	casgstatus(gp, _Grunning, _Grunnable)
	// 使当前 m 放弃 g
	dropg()
	// 并将 g 放回全局队列中
	lock(&sched.lock)
	globrunqput(gp)
	unlock(&sched.lock)

	// 重新进入调度循环
	schedule()
}
```






### 抢占式调度









### 相关方法

Go1.14.15

对应文件：src/runtime/proc.go

* 查找可执行的G： findrunnable()
* 创建新的goroutine: newproc()
* 将goroutine放入本地队列：runqput()
* 将本地队列的goroutine放入全局队列：runqputslow()



### 思考

1. 为什么新创建的goroutine会被放入runnext？



### 相关文章：

《Go语言设计与实现》调度器：https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/《Go Runtime Scheduler》slides：https://speakerdeck.com/retervision/go-runtime-scheduler
用 GODEBUG 跟踪 Go 调度器：https://eddycjy.gitbook.io/golang/di-9-ke-gong-ju/godebug-sched
蚂蚁研究员王益，Go 调度器的隐秘世界：https://zhuanlan.zhihu.com/p/244054940
白明，图解调度器：https://mp.weixin.qq.com/s/mI03sUUaUvZhl7Xb2JPSDg
峰云抢占式调度：http://xiaorui.cc/archives/6535
欧神 协作与抢占：https://golang.design/under-the-hood/zh-cn/part2runtime/ch06sched/preemption/
Go 系统调用：https://wweir.cc/post/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%E5%9C%A8-golang-%E4%B8%AD%E7%9A%84%E5%AE%9E%E8%B7%B5/
系统调用权威指南：https://arthurchiao.art/blog/system-call-definitive-guide-zh
newstack 抢占：https://mcll.top/2019/05/25/%E5%8D%8F%E7%A8%8B%E6%8A%A2%E5%8D%A0/
系统调用的图不错：https://aandds.com/blog/go-scheduler.html#Go-Scheduler-org0000005
刘丹冰【PM 何时创建】：https://segmentfault.com/a/1190000021951119?utm_source=sf-similar-article
GMP 总体+源码：https://segmentfault.com/a/1190000023869478
GPM P 功能比较好：http://odin.show/2020/04/25/golang-GPM/