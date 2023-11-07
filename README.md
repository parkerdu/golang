# Golang学习笔记

# 一、Goland快捷键

1、快速定位到行号 

Ctrl + G

## 查找

1. Ctrl + R 替换文本
2. Ctrl + F 查找文本
3. Ctrl + Shift + F 全局查找



typora快捷键

```
代码块：```+Enter
```



# 二、golang性能分析工具使用

## 2.1 pprof使用

```go
// step1: 在代码开始的地方添加下列代码
import _ "net/http/pprof"

go func() {
    fmt.Println(http.ListenAndServer(":6000", nil)
}()
                
// step2： 在windows上面执行下列命令
go tool pprof -http 0.0.0.0:8888 http://10.30.118.12:6000/debug/pprof/profile

// step3: 若本地保存了文件，可以用下列命令打开
go tool pprof -http 0.0.0.0:8888 本地文件路径
```



## 2.2 trace使用

```go
// step1: 在代码开始的地方添加下列代码
import _ "net/http/pprof"

go func() {
    fmt.Println(http.ListenAndServer(":6000", nil)
}()

// step2: 下载得到trace.out， 在win上执行下列命令
wget -O trace.out hhttp://10.30.118.12:6000/debug/pprof/trace

// step3: 分析
go tool trace trace.out
```



# 三、垃圾回收

## 3.1 标记清除法

标记清除（Mark-Sweep）算法是最常见的垃圾收集算法，标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记（Mark）和清除（Sweep）两个阶段：

1. 标记阶段 — 从根对象出发查找并标记堆中所有存活的对象；
2. 清除阶段 — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表；

<span style="color:red;background:;font-size:12小;font-family:字体;">问题：在整个标记清楚过程中，用户程序还是不可执行，因为需要获取每个对象的存活状态。</span>

一言以蔽之：STW(Stop the world), 并且清楚那些从根节点不可达的对象。

# 四、三色标记法（go1.5）

白色标记

灰色标记

黑色标记

初始条件，所有对象均为白色

步骤一：比哪里root，将所有root标记为灰色

步骤二：将所有灰色对象的下一个标记为灰色，并将步骤一中的灰色对象标记为黑色

步骤三：重复步骤二，直到没有灰色对象。

最后会只剩黑色和白色，白色即可需要回收的垃圾对象。

<span style="color:GREEN;background:;font-size:14;font-family:字体;">如果三色标记过程不启用STW</span>

若在

参考链接：https://www.yuque.com/aceld/golang/zhzanb

# 五、调度器的生命周期

1、M0

启动一个进程后的第一个线程，将其编号为0，记位M0

负责执行初始化操作（init）和启动第一个G(例如main函数，就是通过M0来创建启动的)

每个进程只存在一个M0 

2、G0

启动一个M后，会马上创建第一个goroutine,它就是G0

G0仅用于负责调度其他G

G0不执行任何用户写的函数

每个M都有一个自己的G0（例如上面的M0, 它也有自己的G0）

在调度或系统调用时，M会切换到G0来调度

<span style="color:GREEN;background:;font-size:14;font-family:字体;">问题：M又是谁来创建的</span>

因此在一个程序运行时候，会有下列过程：

启动程序—>进程启动—>创建M0—>创建M0自己的G0—>创建GOMAXPROCES个P—>创建main函数的Goroutine—>M0绑定一个P并将goroutine-main放入本地队列进行执行—>执行main函数（main函数又可以生成其他goroutine）

<span style="color:GREEN;background:;font-size:14;font-family:字体;">问题：init()函数由谁来执行的？</span>



源码分析

M从全局队列获取G

```GO
// Try get a batch of G's from the global runnable queue.
// sched.lock must be held.
func globrunqget(_p_ *p, max int32) *g {
	assertLockHeld(&sched.lock)
	// 若全局队列为空，返回nil
	if sched.runqsize == 0 {
		return nil
	}
	
    // 1、n = 全局队列/最大P(默认为cpu处理器个数) + 1
    // 这里 sched.runqsize和gomaxprocs都是int32，所以相除结果是个向下取整的int32, 例如3/6=0, 因此n最小取值为1
	n := sched.runqsize/gomaxprocs + 1
	if n > sched.runqsize {
		n = sched.runqsize
	}
    // 2、如果传了max，则n不可大于max
	if max > 0 && n > max {
		n = max
	}
    // 3、n可大于P本地队列的一半，P本地队列容量为256的数组，所以n不可大于128
	if n > int32(len(_p_.runq))/2 {
		n = int32(len(_p_.runq)) / 2
	}

	sched.runqsize -= n

	gp := sched.runq.pop()
	n--
	for ; n > 0; n-- {
		gp1 := sched.runq.pop()
		runqput(_p_, gp1, false)
	}
	return gp
}
```



和刘丹冰讲的场景4中的源码：

### (4)场景4

G2在创建G7的时候，发现P1的本地队列已满，需要执行**负载均衡**(把P1中本地队列中前一半的G，还有新创建G**转移**到全局队列)

```
（实现中并不一定是新的G，如果G是G2之后就执行的，会被保存在本地队列，利用某个老的G替换新G加入全局队列）
```

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1650776570176-d9d5abd4-3a48-461c-a43c-6ef504c4038f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_32%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



这些G被转移到全局队列时，会被打乱顺序。所以G3,G4,G7被转移到全局队列。

<div style="color:red;background:;font-size:14;font-family:字体;">使用go func() 关键字新开一个goroutine时候，新开一个goroutine时候，流程为：</br>
    1、编译阶段将go关键字转换为runtime.newproc函数调用</br>
    2、在runtime.newproc中会先初始化一个G结构体（通过gFree获取或者新创建一个，见《go语言设计与实现》p263），再将这个G放入运行队列</br>
    3、将这个G放入运行队列是调用了runtime.runqput方法（见《go语言设计与实现》p264）</br>
    	runtime.runqput有一个参数为next bool</br>
		当该参数为true时候，会将这个G放到P的runnext上面，作为P执行的下一个G
		当该参数为false时，才会将G放到P的本地队列runq[256]guintptr中，此时当G本地队列满了，会调用runtime.runqputslow将本地队列中的一半以及新创建的G放入全局队列

会优先加到当前P的本地队列中，若本地队列已满则将其放入全局队列</div>

上述场景四即对应源码runtime.runqput

```go
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}
	// 一、next为true时候，将gp加入设置到p中的runnext，作为P下一个执行的G
	if next { 
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}
	// 二、next为false时候，如果p本地队列还有空间则加入本地队列
retry:
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
    // 如果p本地队列还有空间则加入本地队列
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
    // 否则加入全局队列，详细代码见下面, 走到这里说明p本地队列是满的共256个，所以将其一半128个加上这个
    // 新g, 一起加入全局队列
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}
```

```go
// Put g and a batch of work from local runnable queue on global queue.
// Executed only by the owner P.
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
	var batch [len(_p_.runq)/2 + 1]*g // 定义一个数组，大小为[128+1]*g

	// First, grab a batch from local queue.
	n := t - h   // 由于p本地队列是满的，所以n=256
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

	if randomizeScheduler { // 打乱原先的顺序
		for i := uint32(1); i <= n; i++ {
			j := fastrandn(i + 1)
			batch[i], batch[j] = batch[j], batch[i]
		}
	}

	// Link the goroutines.
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



<iframe height='265' scrolling='no' title='Fancy Animated SVG Menu' src='//codepen.io/jeangontijo/embed/OxVywj/?height=265&theme-id=0&default-tab=css,result&embed-version=2' frameborder='no' allowtransparency='true' allowfullscreen='true' style='width: 100%;'>See the Pen <a href='https://codepen.io/jeangontijo/pen/OxVywj/'>Fancy Animated SVG Menu</a> by Jean Gontijo (<a href='https://codepen.io/jeangontijo'>@jeangontijo</a>) on <a href='https://codepen.io'>CodePen</a>.
</iframe>



总结创建goroutine过程：

<pre style="color:yellow;font-size:18px">对刘丹冰视频中的补充</br>
视频中老师对于go func()过程的创建只是简单带过(详细过程可以书上6.5.4节的源码)，说创建了一个goroutine, 并且在放入运行队列时候说到，如果本地队列满了，会放到全局队列，这种说法是不准确的，其实不仅将该g放入全局，还有P本地队列的一半</pre>

参考视频：https://www.bilibili.com/video/BV19r4y1w7Nx?p=5&vd_source=d9290e9cb4c92234e818175dda09040e

<pre style="color:green;font-size:18px">我对goroutine创建的总结：</br>
    首先我们go func创建一个goroutine，在编译期间会将这个关键字go转换为runtime.newproc函数, 在newproc函数完成两件事情：</br>
	一个是获取或创建一个goroutine结构体，也就是runtime.g
		获取就是从P的缓存（runtime.p中gFree字段）中获取，这里面的goroutine状态为Gdead,如果P中没有就从调度器的缓存中获取（sched.gFree）
		如果P本地换行和调度器都获取不到，才会去创建一个新的g
		最后新的g初始化了之后，会将go func传入的参数移动到g的栈上，并且切换g状态为Grunnable
	二是将新的g放到运行队列上
		该步骤是调用runtime.runqput方法将newg放到运行队列上，会优先将新的g放到p的本地队列中，如果本地队列满了，会调用runtime.runqputslow将P本地队列上的一半以及这个新的g放到调度器的全局队列中
会调用runtime.newproc1函数（这个调用按幼麟实验室的视频来说是在g0的栈上进行的，待学习），new</pre>



## 场景11源码分析

### 场景11



G8创建了G9，假如G8进行了**非阻塞系统调用**。



![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1650777823944-25f0ea1a-3431-457e-b4cf-342654a953b6.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_76%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)



​		M2和P2会解绑，但M2会记住P2，然后G8和M2进入**系统调用**状态。当G8和M2退出系统调用时，会尝试获取P2，如果无法获取，则获取空闲的P，如果依然没有，G8会被记为可运行状态，并加入到全局队列,M2因为没有P的绑定而变成休眠状态(长时间休眠等待GC回收销毁)。

 系统调用的过程：

![](C:\Users\parker\Desktop\golang面试准备\截图\2020-02-05-15808864354688-golang-syscall-and-rawsyscall.png)

在执行系统调用之前会调用runtime.entersyscall完成进入系统用前的准备工作，在系统调用之后会调用runtime.exitsyscall位当前goroutine重新分配资源等工作，下面对这两个过程分别具体分析。

#### 1、准备工作

[`runtime.entersyscall`](https://draveness.me/golang/tree/runtime.entersyscall) 会在获取当前程序计数器和栈位置之后调用 [`runtime.reentersyscall`](https://draveness.me/golang/tree/runtime.reentersyscall)，它会完成 Goroutine 进入系统调用前的准备工作：

```go
func reentersyscall(pc, sp uintptr) {
	_g_ := getg()
	_g_.m.locks++
	_g_.stackguard0 = stackPreempt
	_g_.throwsplit = true // 禁止栈增长

	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
	casgstatus(_g_, _Grunning, _Gsyscall)

	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick
    
    // 将当前M上的pp进行分离
	_g_.m.mcache = nil
	pp := _g_.m.p.ptr()
	pp.m = 0
    // 这里M会记住pp
	_g_.m.oldp.set(pp)
	_g_.m.p = 0
	atomic.Store(&pp.status, _Psyscall) // 将pp状态设置位系统调用
	if sched.gcwaiting != 0 {
		systemstack(entersyscall_gcwait)
		save(pc, sp)
	}
	_g_.m.locks--
}
```

#### 2、恢复工作

runtime.exitsyscall--> 1、runtime.exitsyscallfase ——> 1、若原来的oldp为_Psyscall状态，会调用wirep将G与该P关联

​																						2、否则尝试从全局调度器sched从获取空闲P

​									2、切换至调度器（g0）的 Goroutine 并调用 [`runtime.exitsyscall0`]

​																		runime.exitsyscall0函数最终会调用rume.schedule()触发调度

```go
func exitsyscall() {
	_g_ := getg()

	oldp := _g_.m.oldp.ptr()
	_g_.m.oldp = 0
    // 1、第一条路快速退出系统	调用，要么将该goroutine和原来分离的p绑定，要么从全局调度器获取空闲P
	if exitsyscallfast(oldp) {  
		_g_.m.p.ptr().syscalltick++
		casgstatus(_g_, _Gsyscall, _Grunning)
		...

		return
	}
	
    // 2、切换至调度器的 Goroutine 并调用 [`runtime.exitsyscall0`]，相对慢的退出系统调用
	mcall(exitsyscall0)
	_g_.m.p.ptr().syscalltick++
	_g_.throwsplit = false
}

```

具体看下第一条快速推出的路

```go
func exitsyscallfast(oldp *p) bool {
	_g_ := getg()

	// Freezetheworld sets stopwait but does not retake P's.
	if sched.stopwait == freezeStopWait {
		return false
	}
	
    // 1、若原来的oldp为_Psyscall状态，会调用wirep将G与该P关联
	// Try to re-acquire the last P.
	if oldp != nil && oldp.status == _Psyscall && atomic.Cas(&oldp.status, _Psyscall, _Pidle) {
		// There's a cpu for us, so we can run.
		wirep(oldp)
		exitsyscallfast_reacquired()
		return true
	}
	
    // 2、否则尝试从全局调度器sched从获取空闲P
	// Try to get any other idle P.
	if sched.pidle != 0 {
		var ok bool
		systemstack(func() {
			ok = exitsyscallfast_pidle()
			if ok && trace.enabled {
				if oldp != nil {
					for oldp.syscalltick == _g_.m.syscalltick {
						osyield()
					}
				}
				traceGoSysExit(0)
			}
		})
		if ok {
			return true
		}
	}
	return false
}

```

较慢的路, 通过mcall调用，在g0上运行

```go
func exitsyscall0(gp *g) {
	_g_ := getg()

	casgstatus(gp, _Gsyscall, _Grunnable)  // 切换g状态为_Grunnable
	dropg()  // 移除M和g关联
	lock(&sched.lock)
	var _p_ *p
	if schedEnabled(_g_) {
		_p_ = pidleget()   // runtime.pidleget() 函数从全局调度器sched获取空闲的p
	}
	if _p_ == nil {
		globrunqput(gp)    // 如果没有空闲的P则将该g放入全局队列
	} else if atomic.Load(&sched.sysmonwait) != 0 {
		atomic.Store(&sched.sysmonwait, 0)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
	if _p_ != nil {   // 获取到P，则在该P上执行g
		acquirep(_p_)
		execute(gp, false) // Never returns.
	}
	if _g_.m.lockedg != 0 {
		// Wait until another thread schedules gp and so m again.
		stoplockedm()
		execute(gp, false) // Never returns.
	}
	stopm()
	schedule() // Never returns.   最终会执行调度
}

```



## 5.1 调度的过程

runtime.main->runtime.mstart->schedule()

写了一个main函数，编译后，运行，进程启动后，

两个全局变量：

1、该进程的主线程对应m0,m0上刚开始会运行g0

2、 g0的协程栈在主线程上分配的，刚开始g0运行在m0上

在runtime.main之前有一个调度初始化, 按照gomaxprocess创建p,并且保存到全局变量allp中

3、全局变量allp, 其中的第一个allp[0]会和m0关联起来

所以在main goroutine创建之前GPM的关联为

g0---m0

​        |

​		allp[0]



main goroutine是由汇编生成的，它会调用runtime.newproc函数，生成g， 并放入allp[0]的本地队列

g0---m0

​        |

​		allp[0]  main_g

在main goroutine里面会调用runtime.mstart开启调度循环（也就是runtime.schedule()）, 并且调用runtime.main函数会调用我们写的main.main函数开始执行，执行完之后调用exit函数退出进程

```go
// The main goroutine.
package runtime
func main() {
	g := getg()  // 这个就是g0
    
	fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()   // 执行main.main
	exit(0)  //  main.main执行完成，则调用exit退出进程
	for {
		var x *int32
		*x = 0
	}
}

```

# 六、byte和rune类型区别



1、类型区别：byte = uint8,  rune = int32， 所以byte代表我们常说的1个字节，而rune类型是4个字节

```
int8 范围 [-128, 127]
unit8 范围 [0, 255]

```

2、使用区别：byte一般用来转化ASCII码， rune 用来转化Unicode(UTF-8)

```go
first := "fisrt"
fmt.Println([]rune(first))
fmt.Println([]byte(first))

# 输出结果
[102 105 115 114 116] // 输出结果 [] rune
[102 105 115 114 116] // 输出结果 [] byte


```

可以看到上面的输出毫无区别，但是换成中文后

```go
first := "第一"
fmt.Println([]rune(first))
fmt.Println([]byte(first))

# 输出结果
=== RUN   TestByte
[31532 19968]
[231 172 172 228 184 128]

```

可以看到 rune类型的31532占了3个字节，



那么什么事 ASCII码？ 什么事Unicode呢？

简单来说这俩都是字符集，ASCII码 总共256个， 而Unicode 在其基础上还有其他的字符

神奇的事情，byte也可以打印出中文字符串

```go
uni := []byte{231, 172, 172, 228, 184, 128}
fmt.Println(string(uni))

=== RUN   TestByte
第一

```

