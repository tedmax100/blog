---
title: Map
date: 2020-12-20 22:42:24
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](https://i.imgur.com/87OwWpr.png)

​<!-- more -->
# [Map](https://blog.golang.org/go-maps-in-action)
Map是一種透過Key來取得Value的一種資料結構, 目的是為了快速查找用O(1).
為什麼MAP能這麼快定位到資料是否存在，或資料本身的位置
因為它使用更多資訊來紀錄資料放在哪邊
就像關聯式資料庫的索引，以空間來換取時間 (反正現在記憶體夠大夠便宜XD)
而Key具唯一性,在Map中若Key重複, 會把Value作後蓋前的更新.

Java的話是HashMap, C# & Python則是Dictionary.
Go的map是一張hash table的引用, 它是一個無序的key/value成對的集合.
Key跟Value可以是不同類型, 但在同一個map內的key一定要同一種類型.

## 底層實現
![](https://i.imgur.com/e9PrDti.png)
Go在map的底層實現是透過之前提到的Array+List去實現的.
Go這裡把hash table稱為buckets是個Array.
bucket內每個元素是指向一串List, 每個節點為bmap, 裡面最多放8組key和value.
根據Hash函數獲得key的特徵值, 作為hash key去映射看是對到哪個bucket.
所以查找Hash table只需要O(1).

如果特徵值重複, 表示元素發生碰撞. 碰撞的元素就會放在同一個特徵值的list中.
當bmap因為最多放8組key, 超過多的會放到overflow這裡.
然後同一list的bmap過多的話, 會進行擴容. 盡量避免碰撞發生.


更多詳情能看連結, 有更多大神整理出文章.
> [cch123/map.md](https://github.com/cch123/golang-notes/blob/master/map.md)

# Map格式
```go
map[KeyType]ValueType
```
# 初始化Map
map是內建型別, 所以不需要額外import任何lib.

map是個reference type.
不是透過該2種方式創建的話, 後續在存取上會引生panic錯誤
```go
// make創建
map變數 := make(map[keyType]valueType)
```
```go 
// 直接實例化, 大括號內能給key:value
map變數 := map[keyType]valueType{}
map變數 := map[string]int{"C":5, "B":6}
```
**注意**
```go
// 通過宣告但沒實例化
var colors map[string]string

colors["red"] = "#da1337"

// 會出現error
// siignment to entry in nil map
// 這是因為使用一個沒初始化的map, 其實你得到的是一個指向nil的指標而已.
```

# 操作方法

## 新增元素
```go
map變數[key] = value
```
## 刪除元素
使用delete()
```go
delete(map變數, key)
```

## 查詢取值
查詢分兩類, 一種直接給key; 一種是走訪, 不需要知道key
### Key查找
- 直接取值
```go
v := map變數[key]
```
```go
value := colors["blue"]

// 直接判斷value是否為零值; key不在, 是返回該valuetype的零值
if value != "" {...}
```
- 取值和取得這key是否存在的標誌
```go
v, exist := map變數[key]
```
```go
value, exists := colors["blue"]

// 判斷key是否存在
if exists != "" {...}
```
- 走訪, 就不必知道key了
> [为什么遍历 Go map 是无序的？](https://gocn.vip/article/1704)
```go
for key, value := range map變數 {...}
```
```go=1
package main

import "fmt"

func main() {
	mapDemo := map[int]int{
		1:  1,
		2:  2,
		3:  3,
		4:  4,
		5:  5,
		6:  6,
		7:  7,
		8:  8,
		9:  9,
		10: 10,
	}
	for k, v := range mapDemo {
		fmt.Println("value :", v, ", key : ", k)
	}
}
/*
value : 9 , key :  9
value : 10 , key :  10
value : 6 , key :  6
value : 7 , key :  7
value : 3 , key :  3
value : 4 , key :  4
value : 5 , key :  5
value : 8 , key :  8
value : 1 , key :  1
value : 2 , key :  2
*/
```
疑, 為什麼走訪順序不如預期中的那樣排序. 
雖然前面有提過本質是hash table本來就是無序的.
但小弟稍微提一下, 更多大神有針對這情況作討論.
Go是random key的方式挑選要從哪個開始走訪.
但是有自己一套隨機函數, 讓挑中每個元素的機會都是一樣的.
也是為了避免太常挑到array裡是nil的部份.

- 清空map
抱歉...Go沒有清空的方法, 就是重新make一個新的map.
不必去擔心GC的效率.
```go
map變數 := make(map[keyType]valueType)
```

- 取得map的元素個數
```go
len(map變數)
```
## 基本範例
```go=1
package main

import "fmt"

func main() {
	// 初始化一個key是string, value是int的map
	m := make(map[string]int)

	// 依據key 塞值
	m["k1"] = 7
	m["k2"] = 13

	fmt.Println("map:", m)

	// 依據key取值
	v1 := m["k1"]
	fmt.Println("v1: ", v1)

	// 取得map的長度
	fmt.Println("len:", len(m))

	// 根據key刪除元素
	delete(m, "k2")
	fmt.Println("map:", m)

	// 判斷k2在不再map內
	_, prs := m["k2"]
	fmt.Println("prs:", prs)

	// 從給定的元素直接去實例化map
	n := map[string]int{"foo": 1, "bar": 2}
	fmt.Println("map:", n)
}
/*
map: map[k1:7 k2:13]
v1:  7
len: 2
map: map[k1:7]
prs: false
map: map[bar:2 foo:1]
*/
```

# [Sync.Map](https://golang.org/pkg/sync/#Map)
因為map在併發情況下讀寫map, 並沒保證線程安全.
```go=1
package main

func main() {
	m := make(map[int]int)

    // 一條goroutine 拼命塞值
	go func() {
		for {
			m[1] = 1
		}
	}()
    // 一條goroutine 拼命取值
	go func() {
		for {
			_ = m[1]
		}
	}()
    
    // 無窮迴圈
	for {

	}

}
/*
fatal error: concurrent map read and map write
*/
```
噴錯了!! 因為使用了兩個goroutine併發的讀跟寫, 
在讀取時, 會檢查hashWriting這標記, 如果這標記為true, 產生了race condition.
map內部機制會對這種併發操作進行檢查並提早發現.
當然也能透過加上讀寫鎖, 來保證線程安全. 但這樣效能很差.

Go則是提供了sync.Map, 該結構具有這些特性:
- 無須初始化, 只需要聲明宣告
- 不能透過上面的操作方法, 要用Store/Load/Delete/LoadOrStore/Range來操作
- Lock-free, 採用[CAS演算法](https://www.itread01.com/content/1547173837.html)
- 沒提供len()方法
```go
type Map struct {
	mu Mutex // 給dirty map用的
	read atomic.Value // readOnly, 這本身保證線程安全
	dirty map[interface{}]*entry
	misses int
}
```

# 操作方式 
## Store 儲存一組key跟value
```go
func (m *Map) Store(key, value interface{}) {...}
```
## Load 依靠key來尋找, 如果存在返回值跟true ; 不存在就返回nil跟false
```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {...}
```
## LoadOrStore 依靠key來尋找, 如果存在返回value跟true ; 不存在就新增key跟value, 並返回false
```go
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {...}
```
## Delete 依靠key來刪除
```go
func (m *Map) Delete(key interface{}) {...}
```
## Range 走訪讀取map中元素的key和value傳給函數f
```go
func (m *Map) Range(f func(key, value interface{}) bool) {...}
```
## 基本範例
```go=1
package main

import (
	"fmt"
	"sync"
)

var printMap = func(key, value interface{}) bool {
	fmt.Printf("key: %s, value: %d\n", key, value)
	return true
}

func main() {
	// 初始化sync.map
	var m sync.Map

	// 新增元素
	m.Store("k1", 7)
	m.Store("k2", map[string]int{"k4": 5})
	// 走訪map
	m.Range(printMap)
	fmt.Println("------------")
	// 依據key讀取元素
	v1, _ := m.Load("k1")
	fmt.Println("v1: ", v1)

	fmt.Println("------------")
	// 依照key 刪除元素
	m.Delete("k2")

	// 讀取或是新增元素
	v1, exist := m.LoadOrStore("k1", 8)
	fmt.Printf("v1: %d, exist: %v\n", v1, exist)
	v3, exist3 := m.LoadOrStore("k3", 2)
	fmt.Printf("v3: %d, exist: %v\n", v3, exist3)
	m.Range(printMap)
	fmt.Println("------------")

	_, exist2 := m.Load("k2")
	fmt.Printf("v3  exist: %v\n", exist2)
}
/*
key: k1, value: 7
key: k2, value: map[%!d(string=k4):5]
------------
v1:  7
------------
v1: 7, exist: true
v3: 2, exist: false
key: k1, value: 7
key: k3, value: 2
------------
v3  exist: false
*/
```

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10215194)