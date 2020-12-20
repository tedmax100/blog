---
title: Container 3兄弟-Heap
date: 2020-12-20 22:16:31
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
# [Heap](https://golang.org/pkg/container/heap/)
Heap(堆積)其實是一個Complete Binary Tree(完全二元樹). 
Go的Heap特性是 各個節點都自己是其子樹的根, 且值是最小的.
同個根節點的左子樹的值會小於右子樹.
所以根節點的值是最小的, 位於索引0的位置.
也有另一種是最大的(max heap), 只是Go這裡是最小的(min heap).
定義 : n個元素 k1, k2,...ki...kn, 並且若且唯若滿足下列關係時稱為heap
ki <= k2i, ki <= k(2i+1) 或者 ki >= k2i, ki >= k(2i+1), i = 1,2,3...,n/2
又因為最小(或最大)的值, 取出該值都只要O(1)的時間.

通常該結構是用來實現(priority queue)優先隊列的方法之一. 能對任務工作作優先等級的排序用.
![](https://i.imgur.com/XbheF2e.png)
底層還是以陣列形式表示
![](https://i.imgur.com/9UZ2bdm.png)

Dijkstra's algorithm也是能用Heap做實現.
# Heap Interface
這裡會提到接口interface, 之後會更詳細的介紹interface的部份

只要實現這些接口, 就可以操作heap提供的各種方法了.
可以看得出來heap接口繼承了sort.Interface, 而sort.Interface內又有三個方法需要實現.
繼承後面會有更詳細的部份介紹.
總之就是要實現這5個方法就行了.

```go
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}

// sort.Interface
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```
## 初始化Heap
```go
heap.Init(customizeHeap)
```

## Heap內建的操作方法
```go
// 一個滿足以上全部接口的堆積結構, 在操作前都要先執行Init()做初始化排序. 
// 複雜度O(log n), n = Len(), 因為是二元搜尋樹的查找
func Init(h Interface) {
	// heapify
	n := h.Len()
	for i := n/2 - 1; i >= 0; i-- {
		down(h, i, n)
	}
}

// 對Array增加一個新元素在最後面
// 並透過up()重新排序把元素作上升, 來滿足min heap的要求.
// 複雜度O(log n), n = Len()
func Push(h Interface, x interface{}) {
    h.Push(x) // 會呼叫我們自定義好的Push()
	up(h, h.Len()-1)
}

// 刪除並且返回Len()-1位置的元素(Array最後一個的元素)
// 等同於對Array做了取[:n-1]的動作, 等於是把第一個元素跟最後一個做了互換後, 透過down(), 把新的根節點下沉到適合的位置, 用來滿足min heap的要求.
// Pop()跟Remove(h, 0 )是一樣的
func Pop(h Interface) interface{} {
	n := h.Len() - 1
	h.Swap(0, n)
	down(h, 0, n)
	return h.Pop()  // 會呼叫我們自定義好的Pop()
}

// 如果heap中有元素的值被修改, 則透過Fix()重新排序, down() & up()也會被呼叫.
// 複雜度O(log n), n = Len()
func Fix(h Interface, i int) {
	if !down(h, i, h.Len()) {
		up(h, i)
	}
}

// 刪除heap中第i個元素, 並且重新排序
// 複雜度O(log n), n = Len()
func Remove(h Interface, i int) interface{} {
	n := h.Len() - 1
	if n != i {
		h.Swap(i, n)
		if !down(h, i, n) {
			up(h, i)
		}
	}
	return h.Pop()
}

// 把元素下沉到對應的子樹合適的位置上
func down(h Interface, i0, n int) bool {...}

// 把元素上升到對應的子樹合適的位置上
func up(h Interface, j int) {...}
```



## 實現自定義的int Heap
首先定義一個類型或是結構, 並且實現那5個方法.
取[官網的範例](https://golang.org/pkg/container/heap/)來說明
```go=1
package main

import (
	"container/heap"
	"fmt"
)

// An IntHeap is a min-heap of ints.
type IntHeap []int

// 返回元素個數
func (h IntHeap) Len() int           { return len(h) }
// 比較大小, 只要索引i的元素<索引j的元素, 就會返回true, 否則返回false, 因為是Min Heap, 所以都在比小
// Max Heap就是反過來比大, 但這方法名還是叫Less不能改就是了XD
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
// 交換h[i]跟h[j]的元素, Golang對swap的寫法很簡單, 不必在創建temp變數在那裡賦值.
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

// 新增元素
func (h *IntHeap) Push(x interface{}) {
	*h = append(*h, x.(int))
}

// Pop出最後一個元素
func (h *IntHeap) Pop() interface{} {
	old := *h
	n := len(old)
    // 把最後一個賦值給x
	x := old[n-1]
    // 建立一組新的slice , 取原有的slice 開始到n-1個元素,  並賦值
	*h = old[0 : n-1]
	return x
}

func main() {
	h := &IntHeap{2, 1, 5}
	heap.Init(h)
	heap.Push(h, 3)
	heap.Push(h, 4)
	heap.Push(h, 9)
    
	fmt.Printf("minimum: %d\n", (*h)[0])
    // first :  1

	for h.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(h))
	}
    // 2 3 4 5 9 
    
    // 把上面走訪heap整段註解掉
    // 修改第1個元素的值
     // 會把原本h[1]的元素, 移動到適當的位置去
    (*h)[1] = 6
    // 讓heap重新排序
    heap.Fix(h, 1)
    
    for h.Len() > 0 {
        fmt.Printf("%d ", heap.Pop(h))
    }
    // 2 4 5 6 9
}
```


## 實現Priority Queue
[官網範例](https://golang.org/pkg/container/heap/)
```go=1
package main

import (
	"container/heap"
	"fmt"
)

// 元素結構
type Item struct {
	value    string  // 元素的值
    priority int   // 元素的優先權值
	index int // 紀錄索引值
}

// PriorityQueue, 本質上是一個*item的Array
type PriorityQueue []*Item

// sort.Interface的實現
// 返回元素個數
func (pq PriorityQueue) Len() int { return len(pq) }

// 因為希望Pop出來的是priority值最大的元素, 所以這裡的邏輯是反著寫
// 其實這就是個Max Heap, 根節點的priority的值大於其他.
func (pq PriorityQueue) Less(i, j int) bool {
	return pq[i].priority > pq[j].priority
}

// 交換pq[i]跟pq[j]的元素, 這裡還要互換兩個元素彼此的index
func (pq PriorityQueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}

// heap.Interface的實現
// 新增元素在Array最後
func (pq *PriorityQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*Item) // 這裡用類型斷言, 日後會補充
	item.index = n // 設定新增進來元素的index
	*pq = append(*pq, item)
}

// Pop出Array最後1個元素
func (pq *PriorityQueue) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	old[n-1] = nil   // 把元素設置為沒有指向任何東西, 等GC來回收原來指向所配置出來的空間
	item.index = -1 // 保險起見, 把pop出去的元素index設置成-1
	*pq = old[0 : n-1] // 從old slice來取 0 ~ n-1的元素來形成新的slice, 並賦值給*pq
	return item
}

// 更新元素的值和優先權, 並且重新排序
func (pq *PriorityQueue) update(item *Item, value string, priority int) {
	item.value = value
	item.priority = priority
	heap.Fix(pq, item.index)
}

func main() {
	items := map[string]int{
		"banana": 3, "apple": 2, "pear": 4,
	}

	pq := make(PriorityQueue, len(items))
	i := 0
	for value, priority := range items {
		pq[i] = &Item{
			value:    value,
			priority: priority,
			index:    i, // 依照清單個數, 依序給index
		}
		i++
	}
    // 初始化Heap
	heap.Init(&pq)


    // 新增一個新元素
	item := &Item{
		value:    "orange",
		priority: 1,
	}
	heap.Push(&pq, item)
    // 修改該元素
	pq.update(item, item.value, 5)

	// 依序Pop出來
	for pq.Len() > 0 {
		item := heap.Pop(&pq).(*Item)
		fmt.Printf("%.2d:%s ", item.priority, item.value)
	}
}
// 05:orange 04:pear 03:banana 02:apple
```
初始化完成時的heap跟Array
![](https://i.imgur.com/p4mZCto.png)  

新增orange, 並修改優先權後, 明顯orange被上升到合適的位置了
![](https://i.imgur.com/Ddk49GY.png)

依序Pop出來
![](https://i.imgur.com/aozYNyB.png)
![](https://i.imgur.com/SrsXGl9.png)  
![](https://i.imgur.com/rbpHAZ4.png)

> 我發現Array位置沒改對...原諒我, 懶得修圖了 - .-
> 但二元樹是對的!'
> [LeetCode 23 ](https://leetcode.com/problems/merge-k-sorted-lists/)可以嘗試用heap來實現
> 小弟我日後補上

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10214861)