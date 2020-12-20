---
title: Type & Struct, 從單細胞生物, 來到多細胞生物了
date: 2020-12-20 22:44:21
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
Type & Struct, 從單細胞生物, 來到多細胞生物了
可以創建類似Java的Class以及宣告屬性與方法
​<!-- more -->
# Type
type這關鍵字用來聲明宣告一些東西
- struct 
等下就介紹
- interface
下次介紹
- 基礎型別
```go=1
package main

import (
	"fmt"
)

// 宣告別名
type name = string

// 定義新的基礎型別
type newStr string

func SayName(str name) {
	fmt.Println(str)
}

func Say(str newStr) {
	fmt.Println(str)
}

func main() {
	var str = "test"
	SayName(str)
	// 這行會噴型別錯誤, 註解掉用下面的方式寫
	// Say(str)

	var ns newStr 
	ns =  "test newStr"
	Say(ns)
}

/*
main.go:25:6: cannot use str (type string) as type newStr in argument to Say
str是字串類型, 可以傳入也是string但卻是別名的SayName, 可見類型一致.
但透過type宣告出來的基礎型別, 卻是不同的類型, 無法傳入使用string的Say.
*/
```
- 類型查詢
```go
//在switch使用變數名稱.(type), 查詢變數是由哪種類型賦值的
switch v := a.(type) {
	case string:
		fmt.Println("string type")
	case int:
		fmt.Println("int type")
	default:
		fmt.Println("other type", v)
}
```
# Struct 
Struct(結構體)是類型中帶有屬性成員的複合類型.
其實就非常類似其他語言的Class (87%相似)
用結構體名稱和結構體屬性來描述真實世界的實體和實體對應的各種屬性.
- 每個屬性必須要有自己的類型和值
- 屬性名稱在結構體內必須唯一
- 屬性的類型也可以是結構體, 或是自己所在的結構體的指針, 但不能跟是自己的類型. 
- 可以屬性都不要設置, 稱為empty struct, 能用來給channel發訊號用.
- 屬性成員名稱小寫開頭為private,  大寫為public
- 沒有繼承, 用的是組合這概念, 這部份更多應用明天分享.

```go
type 類型名稱 struct {
    屬性1 屬性1類型
    屬性2 屬性2類型
    屬性3, 屬性4, 屬性5 屬性345類型 (需要相同類型)
    類型  // 匿名屬性, 類型名稱就是成員屬性名稱
    ...
}
```

## 初始化
有很多種方式...這裡有沒有列出全部, 我也不太清楚QQ
JS要建立一個object, 也是超多種方式XD
```go
// 匿名結構體, 無須透過type關鍵字來定義
p := struct {
    X int
    Y int
} {
    X : 20,
    Y : 10,
}
```
```go
// 透過var聲明
type Point struct {
  X int
  Y int
}

var p Point
p.X = 20
p.Y = 10
```
```go
// 透過var的簡短聲明
var p = Point{
    X: 20,
    Y: 10,
}
```

```go
// 透過new實例出結構體,p是一個Point指標類型, 指向Point結構體的實例. 
p := new(Point)
p.X = 20
p.Y = 10
```
> new()的方法介面 : 回傳的就是指向該類型的指標
> func new(Type) *Type

```go
// 因為沒有類別也沒多載, 所以用各種不同名稱的外部方法來模擬建構式
func NewEmptyPoint() Point {
	return Point{
	}
}
func NewPoint(x, y int) Point {
	return Point{
        X : x,
        Y : y,
	}
}

func NewEmptyPointPtr() *Point {
	return &Point{
	}
}
func NewPointPtr(x, y int) *Point {
	return &Point{
        X : x,
        Y : y,
	}
}
```
```go=1
// demo/pointer.go
package pointer

type Point struct {
  X int
  Y int
}

func New(x, y int) Point {
	return &Point{
        X : x,
        Y : y,
	}
}

// main.go
package main
import "demo/pointer"

func main() {
    // 這樣有沒有比較像建構式的feel了
    p := pointer.New(10, 20)
}
```

這裡會發現跟C有些不同了, C對於ptr類型需要用->來存取成員屬性.
Go施予了語法糖來方便開發者, 自動的把ptr類型的p.X轉成(*p).X 


## Pointer to Struct vs  Struct value
上面會發現struct在使用上會有pointer to struct(結構體指針)跟Struct value(結構體實例)2種類型.

- 結構體指針 
    - 一個指向結構體實例的ptr
    - 傳遞給函數當參數時, 就只會複製該ptr而已, 省很多記憶體, 也快速.
    - 對結構體指針作任何修改, 都會影響到該指針所指向的結構體去作修改.
    - 要直接操作指向的對象時,要加上*
    - 會發生逃逸現象, 需要透過GC來回收.
    - 結構體指針的空值都是nil
- 結構體實例
    - 傳遞給函數當參數時, 會複製物件本身.
    - 傳遞給函數時, 會放在stack內; 在離開函數時, 會被釋放.

```go=1
package main

import "fmt"

type Bag struct {
	items []int
}

func Insert(b *Bag, itemId int) {
	fmt.Printf("address of *b: %p\n", b)
	b.items = append(b.items, itemId)
}

func InsertValue(b Bag, itemId int) Bag {
	fmt.Printf("address of b: %p\n", &b)
	b.items = append(b.items, itemId)
	return b
}

func main() {
	bag := new(Bag)
	fmt.Printf("address of bag: %p\n", bag)
	fmt.Println("新增元素前給ptr: ", bag)
	Insert(bag, 1000)

	fmt.Println("新增元素後給ptr: ", bag)

	bagValue := Bag{}
	fmt.Printf("address of bagValue: %p\n", bag)
	fmt.Println("新增元素前給實例前: ", bagValue)
	InsertValue(bagValue, 1001)
	fmt.Println("新增元素後, 但沒賦值回去: ", bagValue)
	bagValue = InsertValue(bagValue, 1001)
	fmt.Println("新增元素後, 有沒賦值回去: ", bagValue)
}

/*
address of bag: 0xc00000c080
新增元素前給ptr:  &{[]}
address of *b: 0xc00000c080
新增元素後給ptr:  &{[1000]}
address of bagValue: 0xc00000c080
新增元素前給實例前:  {[]}
address of b: 0xc00000c100
新增元素後, 但沒賦值回去:  {[]}
address of b: 0xc00000c140
新增元素後, 有沒賦值回去:  {[1001]}
*/
```
看完輸出能發現, 透過指針傳遞的都是指向同一個位置的變數, 我們對它作操作, 在方法結束後, 他的改變都是有效的.
透過值傳遞, 都不是同一個變數, 都是透過複製出來的副本, 所以要透過回傳, 再把回傳值複製一份給外面, 不然就不會真的作到修改.

# 結構體方法
Go中的方法, 適用於特定類型的函數. 稱為Receiver(接收器)
如果該特定類型是結構體實例或者是結構體指針時. 
接收器的概念就類似JS的this. 就是方法作用的目標!!
當然任何類型都可以有自己的方法.
```go
// (b *Bag) 這個就是接收器, 接受來自Point類型的指標
func (b *Bag) Insert(itemId int) {...}

// (b Bag) 這個就是接收器, 接受來自Point實例
func (b Bag) Insert(itemId int) {...}
```

### 接收器的命名
[官方建議receiver的名字](https://github.com/golang/go/wiki/CodeReviewComments#receiver-names), 第一個字小寫, 而不是用self/this等命名.

### 接收器的類型
選擇在結構體方法的接收器是要用值還是指標...
蠻難抉擇的, 大部分都是用指標.
只有小部份情形會用值傳遞.

- map, func, chan 其實引用類型(reference type), 都是指針了,別再用一個指針指向他們, 然後作操作.
- 如果結構體內有sync.Mutex或其他跟同步相關字眼的, 也別傳值, 傳指針, 讓各地方都用同一個記憶體空間作同步操作.
- 如果想要呼叫的函數, 就直接能作內容修改, 就傳指針
- 如果是自定義的結構體、Array、Slice就傳指針, 不用多複製; 且意圖更明顯, 就是在操作該物件自己; [官方建議](https://github.com/golang/go/wiki/CodeReviewComments#receiver-type)如果Array容量很小還是傳值比較好, 但我自己不太清楚怎樣去定義"小", 所以我還是都傳指針.
- 如果是基礎型別或者是內建的型別(time.Time這種), 它內部沒有指針屬性或者沒有mutable屬性時, 就傳值, 就不會發生逃逸進到Heap等待GC.
- 不清楚? 就是傳指針

但又如何XD 
反正Go其實就只有傳值這概念, 只是傳的如果是指針類型, 還是複製一份指針的副本.
上面有提到會把ptr類型轉成(*ptr), 直接指向該物件去操作.
所以官方才說不清楚判斷該傳什麼, 就傳指針.

我們要清楚的是, 該類型到底是基礎型別還是引用類型(), 這2種都傳值 
裡面有沒有同步需要用到的mutex這些, 有就是傳值,
其他都傳ptr 就行了.

#### 引用類型的範例
```go=1
package main

import "fmt"

func PrintMap(m map[string]int) {
	fmt.Printf("address of map: %p\n", m)
}

func PrintFunc(f func()) {
	fmt.Printf("address of func: %p\n", f)
}

func PrintChan(c chan int) {
	fmt.Printf("address of chan: %p\n", c)
}

func PrintSlice(s []int) {
	fmt.Printf("address of slice: %p\n", s)
}

func PrintArray(a [3]int) {
	fmt.Printf("address of array: %p\n", &a)
}

func PrintArrayPtr(a *[3]int) {
	fmt.Printf("address of array: %p\n", a)
}

func main() {
	m := make(map[string]int)
	fmt.Printf("address of map: %p\n", m)
	PrintMap(m)

	fun := func() {
		fmt.Println("func")
	}
	fmt.Printf("address of func: %p\n", fun)
	PrintFunc(fun)

	channel := make(chan int)
	fmt.Printf("address of chan: %p\n", channel)
	PrintChan(channel)

	s := make([]int, 10)
	fmt.Printf("address of slice: %p\n", s)
	PrintSlice(s)

	// Array不是引用類型
	a := [3]int{1, 2, 3}
	fmt.Printf("value of array: %p\n", a)
	fmt.Printf("address of array: %p\n", &a)
	PrintArray(a)
	PrintArrayPtr(&a)
}
/*
address of map: 0xc000078150
address of map: 0xc000078150
address of func: 0x489520
address of func: 0x489520
address of chan: 0xc000076060
address of chan: 0xc000076060
address of slice: 0xc0000200f0
address of slice: 0xc0000200f0
value of array: %!p([3]int=[1 2 3])
address of array: 0xc000018560
address of array: 0xc0000185a0
address of array: 0xc000018560
*/
```
很明顯這些都是引用類型, 我們操作的一直都是指針類型的變數, 
就不必再用一個指針去指向它們了.
> 後面的0xnnnnnn數字不同, 每次跑我也都不同, 那是記憶體開始位置, 每次都會不同的, so...跑出來跟我範例不同, 不是程式寫錯QQ

> [[Go 語言教學影片] 在 struct 內的 pointers 跟 values 差異](https://blog.wu-boy.com/2019/05/what-is-different-between-pointer-and-value-in-golang/)
> 這是AppleBoy大大的影片, 有提到goroutine內傳指標會出現的問題.

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10215377)