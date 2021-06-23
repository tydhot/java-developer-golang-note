# 概览
Goalng中通过sync.pool提供了对象池的实现来达到对象复用的目的。在netty中，也通过Recycle类实现了类似的对象池实现。在netty的对象池Recycle中，当A线程需要将B线程申请的对象回收到对象池中的时候，会专门开辟一个专门由A线程回收到B线程的队列，以避免回收对象的时候所发生的资源竞争。类似的，在golang的对象池sync.pool中也是通过类似的思想来实现所要达到的目的。
# sync.pool的结构
````
type Pool struct {
	noCopy noCopy

	// local为下面的poolLocal结构体所组成的固定大小的数组，具体的大小为P的数量
	local     unsafe.Pointer
	// local数组size
	localSize uintptr       

	// victim与local相同，也是poolLocal结构体所组成的固定大小的数组，具体的大小为P的数量
	// 当一次gc发生的时候，将会把local中的元素转移到victim中，原本victim中的元素将会被回收
	victim     unsafe.Pointer 
	// victim数组size
	victimSize uintptr        

	// 当对象池中不存在对象的时候的初始化对象方法
	New func() interface{}
}

type poolLocalInternal struct {
	// 对象中缓存的一个对象，为每个P所私有
	private interface{} 

	// 当当前P将超过一个对象放入对象池的时候将会把后续对象放入这个队列，只有当前
	// P可以将对象从队列头部插入或者获取数据，其他P只能从尾部获取
	shared  poolChain   
}

type poolLocal struct {
	// 数组中某个P对应的数组元素
	poolLocalInternal

	// 防止缓存行失效而导致伪共享的偏移量，在java的Disruptor中也使用了类似的技巧
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
````
在结构上，local数组为每个P都分配了一个位置，当固定的P将对象放入池中，或者从池中获取对象的时候，一开始都会围绕其所对应的槽位操作。这里的思路是和netty的对象池Recycle类似的，通过空间换时间的方式尽可能的避免资源竞争。
# sync.pool的初始化
在sync.pool中初始化的逻辑能够更清晰的了解其内部的逻辑。但是首先需要看几个变量。
````
var (
	allPoolsMu Mutex

	// allPools维护着目前所有local中存在元素的pool，需要在stw的时候将这里的所有pool的local
	// 转移到victim中，维护该集合的目的是可以在stw的时候通过注册的清理函数高效执行清理
	allPools []*Pool

	// oldPools维护着目前victim存在数据的pool，在stw的时候通过注册的清理函数可以通过该数组
	// 将所有pool的victim高效清理。
	oldPools []*Pool
)
````
在上面的基础上，可以看到清理函数的具体逻辑。
````
func init() {
	// 在sync.pool初始化的时候，其会向runtime注册清理函数
	runtime_registerPoolCleanup(poolCleanup)
}

func poolCleanup() {

	// 这个方法将会在gc开始时的stw被调用。
	// 首先，其会将oldPools中存在的pool的victim中的元素全部回收
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	// 之后将allPools中所有pool的local中的所有元素全部转移到victim中
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	// 经过上面的处理，原本allPools中的pool的local元素全部到了victim，因此将其设置为
	// oldPools，而原本的oldPools的victim已经被全部清理，不需要再进行维护再其中
	oldPools, allPools = allPools, nil
}
````
# sync.pool中P与对应数组位置的绑定
在sync.pool中，通过pin()方法可以返回P对应的数组元素，如果需要重新指定，也将通过pinSlow()重新完成绑定，这两个方法也是sync.pool中的核心基础方法。
````
func (p *Pool) pin() (*poolLocal, int) {
	// 获取当前P的id，并禁止当前M的P被抢占，由于接下来的所选取的数组的位置是根据P的id选取的
	// 一旦执行的P发生变化，那么可能会导致接下来此处运行的代码将有不同P根据相同id选取的位置而引起冲突
	pid := runtime_procPin()

	s := atomic.LoadUintptr(&p.localSize) 
	l := p.local  
	// 如果当前的pid在当前local数组大小内能够获取并不会越界，那么将直接返回对应位置的元素
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	// 如果是第一次或者在运行过程中P的数量发生了变化，那么需要重新建立local数组
	return p.pinSlow()
}

// 通过数组起始偏移量与pid来获取对应下标的偏移量地址
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
	lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
	return (*poolLocal)(lp)
}

func (p *Pool) pinSlow() (*poolLocal, int) {
	// 由于后面需要加锁，所以这里先解除禁止抢占
	runtime_procUnpin()
	// 加锁，这里也是重新申请local数组存在性能开销的地方，因此方法名叫做pinSlow。
	// 这里的锁是全局锁，所有pool的local初始化都被阻塞。因为在这期间，需要对上文提到的全局的
	// allPools数组进行添加。
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	// 取得锁之后重新禁止抢占
	pid := runtime_procPin()
	// 重新重复pin()方法中的检查操作，在取得锁的过程中，可能local数组已经被初始化完毕
	s := p.localSize
	l := p.local
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	// 如果local数组在这期间仍旧还没有被初始化，那么将该pool添加进allPools，表明在下一次gc
	// 的时候需要对该pool的local进行迁移。由于这禁止抢占的这一事件内不会发生gc，所以这里的
	// 顺序并不会出现问题。
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// 之后根据当前P的数量重新申请相应大小的local数组，保存相关偏移量以便获取。
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
	return &local[pid], pid
}
````
在sync.pool中，通过pin()方法完成了P的id与数组对应下标位置的绑定，通过这一基础函数保证在接下来关于sync.pool的操作可以直接获取当前P私有的那个位置，尽可能避免资源竞争。但是，在第一次初始化local数组的时候，不可避免需要加锁，由于需要修改全局的stw辅助数组，需要添加全局锁，性能开销相对较大。
# sync.pool的Put()
相比底层基础函数的实现，业务逻辑相对会简单的多，尤其是Put()方法的逻辑。
````
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	// 竞态检测相关代码，可无视
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
	// 获取当前P id所绑定的local数组下标元素
	l, _ := p.pin()
	// 如果当前下标的元素private没有对象，那么直接赋值到private上即可
	if l.private == nil {
		l.private = x
		x = nil
	}
	// 如果private已经存在对象，那么直接将其放入到队列的头部
	if x != nil {
		l.shared.pushHead(x)
	}
	// 在之前的pip()中禁止抢占，那么在这里解除
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}
````
# # sync.pool的Get()
````
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
	l, pid := p.pin()
	// 首先直接尝试从private中获取，由于private是当前P私有的，不需要加锁
	x := l.private
	l.private = nil
	if x == nil {
		// 如果private没有对象，那么尝试从当前P的队列的头部获取
		x, _ = l.shared.popHead()
		if x == nil {
			// 如果仍旧没有拿到，那么只能从别的P的队列中尝试捞取一个对象
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
	// 如果上面的逻辑中始终没有获取到对象
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}

func (p *Pool) getSlow(pid int) interface{} {
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// 尝试从别的P对应的队列的尾部偷一个对象
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// 如果仍旧没有捞到对象，那么从victim中尝试获取，总体逻辑和在local中大体获取相同
	// 也是先从victim中获取当前P对应的数组元素，再先尝试从private中获取，否则则会从
	// 各个victim数组的元素的队列尾部尝试获取一个对象（相比local缺少直接从从自己的队列头部获取）
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// 如果仍记没有获取到，则将victim的大小置为0，以便后续不需要再从victim中获取
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}
````
# 总结
sync.pool的实现思路与netty的Recycle实现思路类似，也是通过空间换时间的方式，尽可能在与对象池的操作中避免资源竞争。但是比较特殊，也比较值得注意的是，sync.pool注册了gc的清理函数，理论上一个对象在进入池中没有被再次利用的话将会在两次gc后被回收掉，这也是在使用的过程中需要注意的。这也导致为了在gc的过程中高效遍历所有pool进行清理的话，需要维护当前所有存在的pool来保证。这也导致了当pool重新初始化lcoal的时候，需要添加代价高昂的全局锁来保证并发的安全性。具体的获取的对象过程中，由于每个P都是一开始访问的其私有的数组下标，只有找不到的时候才会从并发队列中获取对象，由于每个P从其私有队列获取对象都是从头部，而别的P从别的队列偷窃对象的时候都是从尾部，因此也在这里尽可能避免了资源竞争的场景。
