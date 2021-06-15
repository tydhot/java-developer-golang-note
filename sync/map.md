Golang sync map
在golang中，线程安全的map实现为sync.Map，相较于java中线程安全的map ConcurrentHashMap，在设计与实现上都有巨大的差别。
## java中的ConcurrentHashMap
java中的ConcurrentHashMap为了实现线程安全，在1.7当中，通过分段锁的实现达到了这一目的，区别于HashTable的全部阻塞操作，分段锁的设计在一定程度上提升了在并发场景下的访问性能。在1.8的过程中，锁的粒度被进一步降低，被缩小到了一个HashEntry首节点的地步，并通过在一定长度的链表后引入红黑树来降低遍历寻找数据的开销。        
但是在golang中的线程安全的sync.Map并没有参考java中的实现，其设计反倒更加类似java中的CopyOnWriteArrayList，但是也不尽然，虽然sync.Map也冗分离了一部分的存储，但是包含更复杂的设计。
## sync.Map的总体结构
````
type Map struct {
	// 对于readOnly的操作都不需要加锁，但是后面所有对于dirty的操作都需要这里的mu加锁。
	mu Mutex

	// 这里的read实际存放的是下面的readOnly结构体，当从map中寻找数据的时候，将会先从
	// readOnly中寻找，在readOnly中的查找都不需要加锁。因为插入数据的时候会先插入到dirty中。
	// 而在readOnly中的操作都是原子操作。
	read atomic.Value // readOnly

	// dirty与readOnly两个map都会在sync.Map存放具体的数据，当map数据插入的时候，将会先
	// 插入到dirty中，直到一定条件后才会晋升到readOnly中。
	dirty map[interface{}]*entry

	// 在从map中根据key寻找数据的时候，先会从readOnly寻找，如果找不到才会从dirty当中寻找，
	// 当在dirty中找到的时候将会misses加一。当misses与dirty的长度相等的时候，将会开始把dirty
       // 中的内容晋升到readOnly中
	misses int
}

type readOnly struct {
	m       map[interface{}]*entry
	
	// 这个值为true的时候说明有存在于dirty，但没有存在在readOnly中的元素。
	amended bool 
}

var expunged = unsafe.Pointer(new(interface{}))

type entry struct {
	// 在map中，元素将以entry的形式存放在map中。当p == nil的时候，该元素已经被删除，同但是
	// dirty中可能还指向该元素。当p == expunged的时候，readOnly中该元素已经被删除，同时dirty中
	// 也没有指向该值。在正常情况下，p为具体存储的值同时dirty中也包含该值。
	// 当一个元素被删除的时候，将会直接将readOnly中p置nil。接下来该值被放入dirty中的时候，将
	// 会把该值置为expunged，并先在dirty中存放具体的值。
	p unsafe.Pointer // *interface{}
}
````
## sync.Map的Store()
sync.Map通过Store()来存储一个键值对。
````
func (m *Map) Store(key, value interface{}) {
	// 在Store()的一开始，先会在readOnly中寻在是否已经存在该key，如果已经存在，尝试通过
	// tryStore()方法对键值对进行更新。
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	// 如果上面直接在readOnly中更新的操作失败，那么需要加锁，将变更操作在dirty中完成。
	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		// 如果在readOnly中，该值已经是expunged，那么先通过cas将其置为nil，代表dirty已经可能存在该值的映射。
		if e.unexpungeLocked() {
			// 该值先前为expunged，说明在ditry中不存在一个非nil的映射，直接更新dirty的值与read相同。
			m.dirty[key] = e
		}
		// 更新key的映射到新的value，因为dirty和readOnly都持有一个e，此时dirty和readOnly都已经更新。
		e.storeLocked(&value)
	// 如果此时dirty已经持有该映射，那么直接修改该key对应的value的地址即可。
	} else if e, ok := m.dirty[key]; ok {
		// 如果该值不存在readOnly，但是在dirty中，那么直接修改dirty的值。
		e.storeLocked(&value)
	} else {
		// 那么在之前的readOnly和dirty中都没有找到，那么就要往dirty中插入。
		if !read.amended {
			// 如果read的amended为false，那么说明此次是第一次往dirty中插入数据，需要
			// 第一次进行dirty的map空间申请。
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		// 往dirty中更新数据。
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}

func (e *entry) tryStore(i *interface{}) bool {
	for {
		p := atomic.LoadPointer(&e.p)
		// 如果为expunged，说明此时dirty中也没有该值的映射，需要先更新dirty。
		if p == expunged {
			return false
		}
		// 如果readOnly中存在没有被删除的值，那么通过cas直接更新具体的值。
		if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
			return true
		}
	}
}
````
## sync.map的Load()
````
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	// 如果能够在readOnly中找到，那么可以直接返回
	if !ok && read.amended {
		m.mu.Lock()
		// 如果在readOnly中没找到，同时dirty中包含readOnly中没有的数据，那么需要加锁从dirty
		// 中找。由于加锁的时候可能已经有元素从dirty中晋升到了read，那么需要双重检测。
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// 如果在dirty中寻找到，那么需要在missLocked()中对misses加一。
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}
	// 当misses大于dirty的长度的时候，将会把dirty的数据全部晋升到dirty中。并将dirty置为nil。
	// 代表当前readOnly已经包含map中的所有数据。
	m.read.Store(readOnly{m: m.dirty})
	m.dirty = nil
	m.misses = 0
}
````
## 总结
总体来说，sync.map通过读写分离，但是并没有像java的CopyOnWriteArrayList完全冗余一个几乎同样大小的数据来保证读写分离，而是将刚插入但是没有几次访问机会的数据隔离在dirty中，保证在执行数据访问的时候尽可能不需要阻塞。但是当需要插入更新readOnly中不存在的数据的时候，则需要对dirty来加锁保证访问，这里的加锁粒度要比java的ConcurrentHashMap要更大，但是不是每次更新都需要加锁，readOnly中的数据可以通过cas保证原子更新。还是如一开始所说，sync.map还是更适合读多写少的场景，当新key的插入十分频繁的时候，性能和普通的map加锁没有太大的区别。