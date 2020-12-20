---
title: Array & Slice
date: 2020-12-20 21:09:04
tags:
---
# Array
![](https://i.imgur.com/9VYZruf.png)
<!-- more -->
```go
// n    陣列元素數量
// type 陣列元素類型
var array變數 [n]type
```
- 長度是固定的, 聲明後無法被改變
- 長度是陣列類型的一部份, 所以兩個長度不同但元素類型相同的陣列, 是不同的類型, ex: [2]int 跟[3]int是不同的類型.
## 初始化方式
```go
a := [3]int{1,2,3} 
b := [...]int{1,2,3,4} //透過初始化給的元素數量來給定長度
c := [3]int{2:100, 1:200} //透過索引初始化元素, 沒被初始化的就是該類型的預設值
d := [...]struct {
    name: string
    age uint8
} {
    { "user1", 5 },
    { "user2", 18 },
}

// 多維度陣列
aa := [2][3]int{{1,2,3}, {4,5,6}}
bb := [...][3]int{{1,2,3}, {4,5,6}} //只有第一個維度能用...
```
## 操作方法
```go
// 取值
data := aa[1] //透過索引取用
//賦值
aa[1] = 2 //透過索引賦值

// 走訪陣列每個元素
for k, v := range d {
    fmt.Println(k, v)
}
/*
0 {user1, 5}
1 {user2, 18}
*/
```

## Array的傳遞
```go=1
package main

import "fmt"

func main() {
	arrA := [2]int{}
	var arrB [2]int

	arrB = arrA

	fmt.Printf("arrA : %p , %v\n", &arrA, arrA)
	fmt.Printf("arrB : %p , %v\n", &arrB, arrB)

	arr(arrA)
}

func arr(x [2]int) {
	fmt.Printf("pass Array : %p , %v\n", &x, x)
}
/*
arrA : 0xc000016100 , [0 0]
arrB : 0xc000016110 , [0 0]
pass Array : 0xc000016150 , [0 0]
*/
```
3個都是[2]int的記憶體位置都不同, 這很明顯Go在Array的賦值和傳參數都是value type,靠複製整個Array的, 因此如果是1億數量的int64陣列, 一個元素佔64bits, 那這陣列就要800MB, 這樣copy 瞬間會需要1.6GB的記憶體空間.

所以也能改成方法參數傳指針, 來避掉這問題.
```go=1
package main

import (
	"fmt"
	"time"
)

func main() {
	arrA := [2]int{1, 2}
	fmt.Printf("arrA : %p , %v\n", &arrA, arrA)

	arr(&arrA)
	arrA[0]++
	fmt.Printf("arrA : %p , %v\n", &arrA, arrA)
}

func arr(x *[2]int) {
	fmt.Printf("pass Array : %p , %v\n", x, *x)
	time.Sleep(time.Second)
	(*x)[0]++
}

/*
arrA : 0xc00008e010 , [1 2]
pass Array : 0xc00008e010 , [1 2]
arrA : 0xc00008e010 , [3 2]
*/
```
會看到都是操作同一個位置的陣列了.
但會引發另一個問題, 原來陣列的指針指向改變了, 函數內的也會更著變動.

這兩個問題, Slice都能有效的處理解決.

# Slice
![](https://i.imgur.com/FPhGTYT.png)

動態分配大小的Array, 可以不必事先指定大小.
雖然是這樣講, 但他其實是一個結構, 透過ptr指向引用的底層Array.
```go
// type 元素類型
// array 指向array的指針
// len 目前slice中有多少元素數量
// cap 可容納多少個元素
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

## 初始化方式
1. 從現有的array或是slice生出新的slice
```go=1
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var arr = [...]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}

	// 取開始到中間的所有元素
	slice0 := arr[:5]
	fmt.Println(slice0)
	// 取中間到尾部的所有元素
	slice1 := arr[5:]
	fmt.Println(slice1)
	// 取中間區間的所有元素
	slice2 := arr[2:7]
	fmt.Println(slice2)
	// 取所有元素, 表示原有的slice
	slice3 := arr[:]
	fmt.Println(slice3)
	// 重置slice, 清空擁有的元素
	slice4 := arr[0:0]
	fmt.Println(slice4)
	fmt.Println("-------------------")
	slice := arr[1:3]
	fmt.Println(reflect.TypeOf(arr))
	fmt.Println(reflect.TypeOf(slice))
	fmt.Println(len(slice), cap(slice))

	fmt.Println(arr)
	fmt.Println(slice)

	fmt.Println("-------------------")
	slice[0] = 0
	fmt.Println(arr)
	fmt.Println(slice)
}

/*
[0 1 2 3 4]
[5 6 7 8 9]
[2 3 4 5 6]
[0 1 2 3 4 5 6 7 8 9]
[]
-------------------
[10]int
[]int
2 9
[0 1 2 3 4 5 6 7 8 9]
[1 2]
-------------------
[0 0 2 3 4 5 6 7 8 9]
[0 2]
*/
```
2. 宣告slice
```go=1
package main

import (
	"fmt"
)

func main() {
	// 宣告字串slice
	var numList []int

	// 宣告一個空slice
	var numEmptyList = []int{}

	fmt.Println(numList, numEmptyList)

	fmt.Println(len(numList), len(numEmptyList))

	fmt.Println(numList == nil)
	fmt.Println(numEmptyList == nil)
}
/*
[] []
0 0
true
false
*/
```
這裡第18行是true, 是因為numList只是宣告, 還沒真正實例化
第19行則是有被實例化被分配到記憶體內了.
因為slice還是個struct動態結構, 所以只能和nil作比較. 

3. 使用make()
```go
// type 元素類型
// size 為slice先分配多少個元素的預設值進去
// cap 預分配的數量, 只是能提前分配空間, 降低之後多次分配空間的效能問題.
make([]type, size, cap)
```
透過make()生成的slice, 一定會實例化配置記憶體,
但透過從其他slice指定開始和結束位置的slice, 只是把新的slice指向舊的slice已經分配好的空間, 只是新的slice註明開始跟結束位子而已, 此時新的slice並不會真的去跟記憶體要一個新的連續空間來宣告新array.
```go=1
package main

import (
	"fmt"
)

func main() {
    // 宣告int slice, 壹開始2個都先分配2元素進去
	a := make([]int, 2)
    // 會發現b, 它的預先配置在記憶體的位置大小, 其實已經是能塞10個元素的配置了
	b := make([]int, 2, 10)

	fmt.Println(a, b)
	fmt.Println(len(a), len(b))
	fmt.Println(cap(a), cap(b))
}
/*
[0 0] [0 0]
2 2
2 10
*/
```

### 透過append()添加元素
append()能為slice動態添加數個元素.
當slice不能容納足夠多的元素時, slice就會進行擴容.
"擴容"往往發生在append()被調用時.
擴容時,容量的擴展規律按照容量的2倍在擴容, 例如1、2、4、8.
```go=1
package main

import (
	"fmt"
)

func main() {
    // 宣告一個len 和cap 都是0的slice
	numbers := make([]int, 0)

	for i := 0; i < 10; i++ {
		numbers = append(numbers, i)
		fmt.Printf("len: %d, cap: %d,  ptr: %p\n", len(numbers), cap(numbers), numbers)
	}
}
/*
len: 1, cap: 1,  ptr: 0xc000016100
len: 2, cap: 2,  ptr: 0xc000016130
len: 3, cap: 4,  ptr: 0xc000018560
len: 4, cap: 4,  ptr: 0xc000018560
len: 5, cap: 8,  ptr: 0xc00001a340
len: 6, cap: 8,  ptr: 0xc00001a340
len: 7, cap: 8,  ptr: 0xc00001a340
len: 8, cap: 8,  ptr: 0xc00001a340
len: 9, cap: 16,  ptr: 0xc00006e080
len: 10, cap: 16,  ptr: 0xc00006e080
*/
```
可以很明顯看到, 當原來的cap滿的時候, 會產生擴容現象.


> 舉個生活例子來說明這len和cap以及擴容.
> 公司發展初期, 資金少, 人員配置也少, 只需要小小的辦公室就能容納所有員工.
> 隨著業務的擴展和收入的增加, 就需要擴編, 但現有辦公室大小是固定的, 無法改變它. 
> 所以公司決定！ 換個更大的辦公室, 每次搬家就要把所有人搬遷到新的辦公處.
> 員工就是slice中的元素
> 辦公室就是配置好的記憶體空間, 大小是固定的
> 搬家就是重新配置
> 不論搬家多少次, 公司名稱都是固定的, 表示外部使用這slice的變數名稱是不會修改的,
> 但因為搬家後地址發生變化, 所以slice內部array指向的地址會有所修改.

```go
// 添加多個元素
numbers = append(numbers, 1, 2, 3)
 
 // 透過令一個slice來添加多個元素
 nums := []int{4,5,6,7}
 numbers = append(numbers, nums...)
```

#### More example
```go=1
package main

import (
	"fmt"
)

func main() {
	a := make([]int, 0, 10)
	b := append(a, 1)
	_ = append(a, 2)
	fmt.Println(b[0])
}
// 2
// 因為b.ptr = a.ptr, 且a的cap有10,足夠插入新元素, 
// 第9行執行完, 會發現a的len還是0
// 執行了 第10行後, 當然append a就會把第0個元素的值給修改掉了.
```
```go=1
package main

import (
	"fmt"
)

func main() {
	a := make([]int, 10, 20)
	b := a[5:]
	fmt.Println(len(b), cap(b))
}
// 5  15
// 因為b等於是對a作重新slice, 只取a的第5到結束的值. 那就是10-5 = 5, 所以len(b)=5
// cap同上, 指針指到的是a.ptr的第5個元素, 20-5= 15
```
```go=1
package main

import (
	"fmt"
)

func doAppend(a []int) {
	b := append(a, 0)
	fmt.Println(b)
}

func main() {
	a := []int{1, 2, 3, 4, 5}
	doAppend(a[0:2])
	fmt.Println(a)
}
// [1 2 0]
// [1 2 0 4 5]
// 調用doAppend時, 傳入2個元素, 但這操作卻把外部的a的第3個元素也改掉了
// 只要把第14行的程式改成doAppend(a[0:2:2])
// [1 2 0]
// [1 2 3 4 5]
// 結果就會正確了, 因為[0:2:2]最後的2就是指定重新切片後的capacity, 這時候指定是2.
// 所以append操作時發現cap >2, 就會重新分配記憶體來存放, 這樣就不會改到原本的了
```
### 透過copy()複製slice到令一個slice
Go內建copy()方法, 可以快速的把slice 作copy
```go
// 回傳有多少個元素被複製過去 
func copy(dst, src []Type) int
```
```go=1
package main

import (
	"fmt"
)

func main() {
	numbers := make([]int, 0)

	for i := 0; i < 10; i++ {
		numbers = append(numbers, i)
	}

	copyA := make([]int, len(numbers))
	fmt.Println("copy cnt:", copy(copyA, numbers))
	fmt.Println("copied data:", copyA)

	copyB := make([]int, 3)
	fmt.Println("copy cnt:", copy(copyB, numbers[2:5]))
	fmt.Println("copied data:", copyB)

	copyC := make([]int, 3)
	fmt.Println("copy cnt:", copy(copyC, numbers))
	fmt.Println("copied data:", copyC)
}
/*
copy cnt: 10
copied data: [0 1 2 3 4 5 6 7 8 9]
copy cnt: 3
copied data: [2 3 4]
copy cnt: 3
copied data: [0 1 2]
*/
```
- copyA宣告的容量是來源的既有元素數量, 所以能完整copy來源所有元素.
- copyB只宣告了3個容量的slice, 之前提過slice可以取開始和結束區間, 這裡用這方式來取值作copy
- copyC一樣容量只有3, 但要複製來源所有元素時, 卻因為容量不夠, 所以沒法複製全部. 又因為擴容只會發生在append, 因此這例子不會自動擴容, 導致後半段資料全被切掉. 

### 刪除slice中的元素
因為slice並沒有提供刪除專用的api.
所以只能用本身特性來刪除元素.
本質操作上就是, 以被刪除的元素位置為分界點, 將該元素的前後兩個部份作拼接.
![](https://i.imgur.com/AONtfse.png)
```go=1
package main

import (
	"fmt"
)

func main() {
	numbers := [...]int{1, 2, 3, 4, 5}
	fmt.Println(numbers)

	index := 2

	fmt.Println(numbers[:index], numbers[index+1:])

	deletedNumbers := append(numbers[:index], numbers[index+1:]...)
	fmt.Println(deletedNumbers)
}
/*
[1 2 3 4 5]
[1 2] [4 5]
[1 2 4 5]
*/
```

> 因為slice如果頻繁刪除新增裡面的元素的話,
> 是會頻繁的搬動位置, 這點對效能損耗較高.
> 可能就要考慮其他資料結構來實做.

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10214513)