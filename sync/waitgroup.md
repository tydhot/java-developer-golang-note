# 概述
在golang中，sync.WaitGroup起到了和java中的CountDownLatch一样的作用，让一批线程（协程）在某个点阻塞，直到某个数量的其他线程（协程）完执行完了其操作才能继续往下执行。       
在java的CountDownLatch中，采用了AQS的共享模式来帮助实现其功能。当需要阻塞的线程通过await()方法阻塞之前需要设定一个初始值count,也就是AQS中的state。当当前线程使用await()阻塞的时候，当相应count数量的线程调用完毕countDown()的时候，才会结束阻塞继续向下运行。当一个线程调用await()的时候，将会由于AQS中的state>0而阻塞挂起在队列当中。而当别的线程调用countDown()的时候，将会给state-1，直到state=0的时候，由于AQS的共享模式，将会将队列中所有阻塞挂起的线程唤醒继续往下执行。
# sync.WaitGroup的结构
````
type WaitGroup struct {
	noCopy noCopy

	// 在老版本的golang中，用一个64位的字段来存放CountDownLatch中的count和阻塞的协程数量
	// 其中高32位存放count。和java一样，这里的count归零后将会唤醒所有阻塞在此的协程。
	// 低32位存放阻塞的协程数量。在java中并没有保存低32位的阻塞数量，
	// 因为在java的AQS中线程都阻塞在队列当中，只需要从队列头部不断向后传播唤醒事件即可。但是
	// 在golang中，通过信号量进行阻塞，那么就需要依靠这个变量来确定执行通过变量的唤醒次数。在
	// 目前版本，这两个字段被分成了2个32位存放在数组中，为了避免在32位编译器下修改64位数据
	// 的非原子性。
	// state1数组的最后一个位置存放的是对协程进行阻塞和唤醒的信号量。
	state1 [3]uint32
}
````
# sync.WaitGroup的Add()
在sync.WaitGroup中，每一次Add()调用后，都会给在sync.WaitGroup的count位根据参数进行相应的增减。（当协程完成其任务准备给sync.WaitGroup的count-1的时候也是通过这个函数传入-1来实现）
````
func (wg *WaitGroup) Add(delta int) {
	// 获取state与信号量
	statep, semap := wg.state()
	// 竞态分析代码，可无视
	if race.Enabled {
		_ = *statep // trigger nil deref early
		if delta < 0 {
			// Synchronize decrements with Wait.
			race.ReleaseMerge(unsafe.Pointer(wg))
		}
		race.Disable()
		defer race.Enable()
	}
	// 对高32位的count与传入的参数进行运算
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	// 目前的count是state的高32位
	v := int32(state >> 32)
	// 低32位w是目前等待的数量
	w := uint32(state)
	if race.Enabled && delta > 0 && v == int32(delta) {
		race.Read(unsafe.Pointer(semap))
	}
	// 如果count数<0直接panic
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	// 在有协程执行Wait()阻塞之后并count归零准备唤醒的时候，不应该再有协程并发通过Add加入
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// 如果目前count仍旧大于0或没有协程阻塞，则直接返回
	if v > 0 || w == 0 {
		return
	}
	// 当代码运行到此处，根据上面的判断必须符合两个条件，count已经为0（目前所有的协程都已经
	// 完成工作）等待的协程数量>0，那么需要准备开始处理唤醒操作。
	// 这里仍旧会获取最新的state与之前的state对比进行一次检查。以防在count归零这一期间
	// 有协程执行wait或者Add与Wait操作之间并发执行。
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// 重制count和等待的协程数量归0
	*statep = 0
	for ; w != 0; w-- {
		// 根据等待的协程数量触发信号量唤醒
		runtime_Semrelease(semap, false, 0)
	}
}
````
# sync.WaitGroup的Wait()
````
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep // trigger nil deref early
		race.Disable()
	}
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		w := uint32(state)
		if v == 0 {
			// 如果目前没有协程通过Add加入，那么就不需要阻塞，直接继续往下
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
		// 如果已经有协程都过add加入🧊未退出，则增加等待的数量并阻塞
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			if race.Enabled && w == 0 {
				race.Write(unsafe.Pointer(semap))
			}
			// 根据信号量阻塞
			runtime_Semacquire(semap)
			// 执行到此处的时候已经被唤醒，state应该已经在Add中被置零此时不应该有别的协程加入
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
	}
}
````

