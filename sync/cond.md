golang cond源码走读
# 概览
Golang中的cond提供了与java juc包中几乎一摸一样的作用，两者的使用和原理基本是一模一样的。
# juc的Condition
在java juc的Condition的使用中，在一开始离不开锁lock的使用，只有持有锁的线程才能在某个condiftion下通过await()方法将锁释放并等到某个线程持有锁后通过这个condition的signal()方法将其唤醒重新进行锁的竞争。每个condition都必须和一个lock绑定，condition的能力都是通过juc的lock所提供。      
当持有锁的线程准备根据某个condition进行等待的时候，将会释放锁并将该线程移入这个condition的私有FIFO队列中进行等待。而另一个线程此时能够竞争到锁而执行起逻辑，当其执行对应condition的signal()方法时候，将会将该condition的队列中的头节点的线程加入lock的AQS等待队列当中尝试重新获取锁，以达到了唤醒的目的。而执行signal()的线程释放锁的时候，原本await而等待并经过signal的线程将会重新竞争到锁而继续执行其剩下的逻辑。
# go中的cond
Golang中的cond的设计思路与java的实现几乎一模一样。
## cond的结构与初始化
````
type Cond struct {
	noCopy noCopy

	// 这里lock对应juc里的lock
	L Locker

	// 这里的notify对应java下Condition的私有等待队列
	notify  notifyList
	checker copyChecker
}

func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}
````
在cond中的两个核心对象在java中都能找到对应的实现，l对应juc的lock，notify对应Condtion的私有等待队列。在NewCond()方法中，你必须传入一个lock才能成功初始化一个cond对象，这与java中对于lock的依赖一模一样，基于cond的状态转变都需要加锁解锁。
## cond的Wait()与Singnal()
````
// 进入Wait()的前提是已经持有了锁
func (c *Cond) Wait() {
	c.checker.check()
	// 加入cond的等待队列
	t := runtime_notifyListAdd(&c.notify)
	// 释放锁
	c.L.Unlock()
	// 在等待队列当中等待
	runtime_notifyListWait(&c.notify, t)
	// 当运行到此处的时候，此时的已经由另一条协程唤醒，开始重新竞争持锁，竞争到之后可以继续往下走
	c.L.Lock()
}

// 进入Signal()的前提是已经持有了锁
func (c *Cond) Signal() {
	c.checker.check()
       // 直接唤醒一个阻塞在等待队列上的另一条协程
	runtime_notifyListNotifyOne(&c.notify)
	// 此时此条协程仍旧持有锁，离开该方法后仍旧需要手动释放锁，其他的协程才能继续走
}
````
# 总结
Java与golang对于cond的设计思路与使用几乎一模一样，都是基于lock与其私有的等待队列来实现，了解了java的实现后能够快速理解golang的cond。
