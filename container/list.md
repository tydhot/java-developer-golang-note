# list概览
list为golang中的双向链表实现，存入list中的元素都会被封装成list中的节点放到双向链表中进行存储。        
简单的list使用代码如下：
````
func main() {
	list := list.New()
	fmt.Println("list length is " +  strconv.Itoa(list.Len()))
	list.PushBack(1)
	list.PushBack(2)
	list.PushBack(3)
	fmt.Println("list length is " +  strconv.Itoa(list.Len()))
	for i := list.Front(); i != nil; i = i.Next() {
		fmt.Println(i.Value)
	}
	for i := list.Front(); i != nil; i = i.Next() {
		if i.Value == 2 {
			list.Remove(i)
		}
	}
	fmt.Println("***************")
	for i := list.Front(); i != nil; i = i.Next() {
		fmt.Println(i.Value)
	}
}
````
在上述代码中，简单的完成了对于list的初始化，元素的添加，list的遍历与元素的移除。可以看到，基于list的遍历都是基于元素的指针进行，这里的操作类似java中的LinkedList。
# list的节点
````
type Element struct {
	next, prev *Element

	list *List

	Value interface{}
}
````
从上面的代码可以看到，在list中，单个节点会被封装成Element存储在队列中，其中next与prev为节点前后的指针，list为该节点所处的list指针，value则来存放具体的值。这里的实现是与java中的LinkedList极为相似的，在java中的LinkedList，每个节点也被封装成了Node存储在链表中。
# list本身的结构
````
type List struct {
	root Element 
	len  int     
}
````        
在List自身的结构体中，root作为根节点，虽然称为根节点，但是在具体的双向链表中，root节点实则将该双向链表连成了一个圈，通过其next可以得到链表中的首节点，而通过其prev则可以得到其尾节点，这里与java的LinkedList维护首尾两个节点的做法不同，更加高效。同时，只需要从根节点的next出发即可完成对于双向链表的遍历，直到通过next到达根节点为止。                
而len则记录了当前双向链表的长度，当需要得到当前链表的长度的时候只需要返回len即可，同样在java的LinkedLis中也通过size记录了当前链表的长度。
# list的初始化
````
func (l *List) Init() *List {
	l.root.next = &l.root
	l.root.prev = &l.root
	l.len = 0
	return l
}
````
list的初始化非常简单，将根节点的前后指针都指向自己，并初始化长度为0即可/
# list的插入与移除
当双向链表中向链表尾部插入元素的时候，如上文所说，由于list中通过根节点的prev一直维护着尾节点，只需要将插入的值包装为list中的节点后插入到根节点的prev之前即可。
````
func (l *List) PushBack(v interface{}) *Element {
	l.lazyInit()
	return l.insertValue(v, l.root.prev)
}

func (l *List) insert(e, at *Element) *Element {
	e.prev = at
	e.next = at.next
	e.prev.next = e
	e.next.prev = e
	e.list = l
	l.len++
	return e
}

func (l *List) insertValue(v interface{}, at *Element) *Element {
	return l.insert(&Element{Value: v}, at)
}
````
同样的，当需要将元素插入到队列的头部的时候，只需要将其包装成节点后插入到根节点的next即可。
而与java的LinkedList不同的是，当需要移除list当中的节点的时候，必须要传入需要移除的list中的节点才行。
````
func (l *List) Remove(e *Element) interface{} {
	if e.list == l {
		l.remove(e)
	}
	return e.Value
}

func (l *List) remove(e *Element) *Element {
	e.prev.next = e.next
	e.next.prev = e.prev
	e.next = nil 
	e.prev = nil 
	e.list = nil
	l.len--
	return e
}
````
相较于java的做法，这样的移除方式虽然存在一定的局限性，但是首先，由于节点本身存在指向所处list的指针，可以避免在错误的队列中移除对应的元素，同时不需要在移除前对整个链表进行遍历（当然在获取该节点的时候已经遍历过了），只需要将前后节点相连并将其next，pre置nil来防止内存泄漏即可。