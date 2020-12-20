---
title: Container 3兄弟-List
date: 2020-12-20 21:26:42
tags:
---
Go有提供幾種 List、Heap、Ring
來依序玩看看
<!-- more -->
# [List](https://golang.org/pkg/container/list/)
![](https://i.imgur.com/uvSvMVU.png)

因為上篇講[Array & Slice](https://ithelp.ithome.com.tw/articles/10214513), 這兩種底層都需要連續的記憶體空間來配置.
List則是可以非連續空間的容器, 也可以支援快速增刪元素.
List由多個節點所組成的, 節點之間透過一些變數紀錄彼此的關係.
且List並沒有限制每個節點的元素類型. 所以可以是任意類型.
但後續轉換時就要注意.

List有多種實現方式 :
- Single Linked List
- Double Linked  List : Go內建這個類型, 相較於single linked list, 在增刪元素時不需要移動元素, 可以原地增刪. 還能夠雙向走訪.
![](https://i.imgur.com/pip4nS1.png)

這是List 的source code, 可以看到有next, prev這兩個ptr, 指向前後各一個元素的位置.
呼叫Init()時, prev, next都指向root節點.

```go
// Element is an element of a linked list.
type Element struct {
	// Next and previous pointers in the doubly-linked list of elements.
	// To simplify the implementation, internally a list l is implemented
	// as a ring, such that &l.root is both the next element of the last
	// list element (l.Back()) and the previous element of the first list
	// element (l.Front()).
	next, prev *Element

	// The list to which this element belongs.
	list *List

	// The value stored with this element.
	Value interface{}
}

// List represents a doubly linked list.
// The zero value for List is an empty list ready to use.
type List struct {
	root Element // sentinel list element, only &root, root.prev, and root.next are used
	len  int     // current list length excluding (this) sentinel element
}

// Init initializes or clears list l.
func (l *List) Init() *List {
	l.root.next = &l.root
	l.root.prev = &l.root
	l.len = 0
	return l
}

// New returns an initialized list.
func New() *List { return new(List).Init() }
```


## 初始化List
```go
// 透過New(), New會去呼叫Init()
變數名稱 := list.New()

// 透過聲明來初始化
var 變數名稱 list.List
```

## 插入新元素
PushFront、PushBack 可以在List的最前面或最後面增加元素.
**PushFront** 是對目前List的root節點前面在多一個元素; 看原始碼會發現呼叫了insertValue(), 第二個參數是root, 然後又呼叫了insert(&Element, root), 第一個參數是新增的元素, 第二個參數是該新增元素要插入在誰的後面, 這裡是安插在root後面. 
**PushBack** 是對目前List的尾巴節點後面多一個元素.

InsertBefore、InsertAfter則是在被標記的元素前或後增加元素.

原始碼
```go
// insert inserts e after at, increments l.len, and returns e.
func (l *List) insert(e, at *Element) *Element {
	n := at.next
	at.next = e
	e.prev = at
	e.next = n
	n.prev = e
	e.list = l
	l.len++
	return e
}

// insertValue is a convenience wrapper for insert(&Element{Value: v}, at).
func (l *List) insertValue(v interface{}, at *Element) *Element {
	return l.insert(&Element{Value: v}, at)
}

// PushFront inserts a new element e with value v at the front of list l and returns e.
func (l *List) PushFront(v interface{}) *Element {
	l.lazyInit()
	return l.insertValue(v, &l.root)
}

// PushBack inserts a new element e with value v at the back of list l and returns e.
func (l *List) PushBack(v interface{}) *Element {
	l.lazyInit()
	return l.insertValue(v, l.root.prev)
}

// InsertBefore inserts a new element e with value v immediately before mark and returns e.
// If mark is not an element of l, the list is not modified.
// The mark must not be nil.
func (l *List) InsertBefore(v interface{}, mark *Element) *Element {
	if mark.list != l {
		return nil
	}
	// see comment in List.Remove about initialization of l
	return l.insertValue(v, mark.prev)
}

// InsertAfter inserts a new element e with value v immediately after mark and returns e.
// If mark is not an element of l, the list is not modified.
// The mark must not be nil.
func (l *List) InsertAfter(v interface{}, mark *Element) *Element {
	if mark.list != l {
		return nil
	}
	// see comment in List.Remove about initialization of l
	return l.insertValue(v, mark)
}
```
```go=1
package main

import (
	"container/list"
	"fmt"
)

func traverse(list *list.List) {
	// 走訪list
	fmt.Printf("root -> ")
	for el := list.Front(); el != nil; el = el.Next() {
		fmt.Printf("%v -> ", el.Value)
	}
}
func main() {
	// 宣告一個List, 並且初始化
	list := list.New()

	// 最後面新增20
	list.PushBack(20)

	// 最前面新增10
	list.PushFront("10")

	// 最後面新增25, 並且保存該新增元素到變數上
	element := list.PushBack(25)

	// 在該元素後面新增26
	list.InsertAfter("26", element)

	// 在該元素前面新增26
	list.InsertBefore(24, element)

	traverse(list)

	fmt.Println("\n---------------------")
	// element 換到第一個元素的後面
	list.MoveAfter(element, list.Front())
	traverse(list)

	fmt.Println("\n---------------------")
	// element 換到第一個元素的前面
	list.MoveBefore(element, list.Front())
	traverse(list)

	fmt.Println("\n---------------------")
	// element 換到最後面
	list.MoveToBack(element)
	traverse(list)

	fmt.Println("\n---------------------")
	// element 換到最前面
	list.MoveToFront(element)
	traverse(list)

	fmt.Println("\n---------------------")
	// 移除該元素
	list.Remove(element)
	traverse(list)
}
/*
root -> 10 -> 20 -> 24 -> 25 -> 26 -> 
---------------------
root -> 10 -> 25 -> 20 -> 24 -> 26 -> 
---------------------
root -> 25 -> 10 -> 20 -> 24 -> 26 -> 
---------------------
root -> 10 -> 20 -> 24 -> 26 -> 25 -> 
---------------------
root -> 25 -> 10 -> 20 -> 24 -> 26 -> 
---------------------
root -> 10 -> 20 -> 24 -> 26 -> 
*/
```

## 走訪List
走訪List需要配合Front()取得第一個元素, 開始往下走訪.
每次就呼叫目前元素的Next(), 只要元素不是nil 就能繼續往下走.
也能逆向往前走, 改用Prev()就可.

## 取得List長度
```go
list.Len()
```

# List vs Slice
比較新增元素、插入元素、走訪的速度
```go=1
package main

import (
	"container/list"
	"fmt"
	"time"
)

func main() {
	t := time.Now()
	sli := make([]int, 10)
	for i := 0; i < 1*100000*1000; i++ {
		sli = append(sli, 1)
	}
	fmt.Println("Slice 新增元素耗費：" + time.Now().Sub(t).String())

	// 比较走訪
	t = time.Now()
	for _ = range sli {
	}
	fmt.Println("走訪Slice耗費:" + time.Now().Sub(t).String())

	// 比較插入元素
	t = time.Now()
	slif := sli[:100000*500]
	slib := sli[100000*500:]
	slif = append(slif, 10)
	slif = append(slif, slib...)
	fmt.Println("Slice 的插入元素耗費 : " + time.Now().Sub(t).String())

	// 比較刪除元素
	t = time.Now()
	index := 100000
	_ = append(sli[:index], sli[index+1:]...)
	fmt.Println("Slice 的刪除元素耗費 : " + time.Now().Sub(t).String())

	sli = make([]int, 10)

	// ---------Slice end, start list
	fmt.Println("------------------------------")

	t = time.Now()
	l := list.New()
	for i := 0; i < 1*100000*1000; i++ {
		l.PushBack(1)
	}
	fmt.Println("List 新增元素耗費: " + time.Now().Sub(t).String())

	t = time.Now()
	for e := l.Front(); e != nil; e = e.Next() {
	}
	fmt.Println("走訪List耗費:" + time.Now().Sub(t).String())

	var em *list.Element
	i := 0
	// 找到1/3處的元素
	for e := l.Front(); e != nil; e = e.Next() {
		i++
		if i == l.Len()/3 {
			em = e
			break
		}
	}
	// 因為是記算插入元素的速度, 所以忽略查找的時間
	t = time.Now()
	l.InsertAfter(2, em)
	fmt.Println("List 的插入元素耗費 : " + time.Now().Sub(t).String())

	// 比較刪除元素
	t = time.Now()
	l.Remove(em)
	fmt.Println("List 的刪除元素耗費:" + time.Now().Sub(t).String())

}
/*
Slice 新增元素耗費：1.749752738s
走訪Slice耗費:35.548381ms
Slice 的插入元素耗費 : 46.402953ms
Slice 的刪除元素耗費 : 92.097862ms
------------------------------
List 新增元素耗費: 17.721431965s
走訪List耗費:364.763942ms
List 的插入元素耗費 : 2.17µs
List 的刪除元素耗費:73ns
*/
```

### 結論
對於資料量很多的情境下, 
如果很頻繁的插入或是刪除, List的成本低到幾乎可以不計算.
但如果頻繁的新增或是走訪查找, Slice的效能高過List許多.

> 首圖是參考[該文章](https://medium.com/backendarmy/linked-lists-in-go-f7a7b27a03b9)的, 該文有講單鏈, 雙鏈跟環鏈, 有機會再分享

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10214704/edit)