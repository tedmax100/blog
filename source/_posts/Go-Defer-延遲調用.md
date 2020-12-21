---
title: Go_Defer_延遲調用
date: 2020-12-21 21:19:31
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
看個例子, 這是一個讀取資料庫取資料的方法
```go=1
func (db *DB) ReadData(age int, results []Result) {
    // 查詢資料庫
    // 錯誤, 釋放連線
    // 取值反射錯誤, 釋放連線
    // 成功, 釋放連線
} 
```
因為GO沒有try{} finally{} 這語句.
所以很多情況如果要在離開函數之前, 作一些必要的動作時
就要在各種case下, 加上處理.
early return的寫法, 也要每個return前都寫一樣的處理, 破壞簡潔.

![](https://i.imgur.com/2R6EfyQ.png)
wtf 很容易寫成這樣 ... 只要邏輯的層數多點的話
​<!-- more -->
But!!!
Go有Defer這延遲載入的語句!!!
剛剛的例子就能夠改成
```go=1
func (db *DB) ReadData(age int, results []Result) {
    // 查詢資料庫
    defer 釋放連線
    // 錯誤
    // 取值反射錯誤
    // 成功
} 
```

來看看defer實際的存放跟執行順序先
![](https://i.imgur.com/IK2asNj.png)
defer 會被後面的執行語句, 依照後進先出LIFO的方式作執行, 

至於defer被觸發的時間點, 就在當前函數返回之前就會被調用.

## defer的結構
```go
type _defer struct {
    siz     int32
    started bool
    sp      uintptr
    pc      uintptr
    fn      *funcval
    _panic  *_panic
    link    *_defer
}
```
fn 存的就是指向defer關鍵字傳入的語句了


```go=1
func main() {
    fmt.Println("begin")
    defer fmt.Print(1)
    fmt.Println("do something")
    defer fmt.Print(2)
    fmt.Println("end")
}
/*
begin
do something
end
2
1
*/
```
也能傳入匿名函數
```go=1
func main() {
	fmt.Println("ithome")

	defer func() {
		fmt.Println("ironman")
		fmt.Println("Day 13 post sucess")
	}()
}
/*
ithome
ironman
Day 13 post sucess
*/
```
進階題 : defer 裡函數裡包著函數
```go=1
func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}

func main() {
	a := 1
	b := 2
    // 記得是FILO
	defer calc("1", a, calc("10", 2, b))

	a = 0
	defer calc("2", a, calc("20", a, b))

	b = 1
}
/*
10 2 2 4
20 0 2 2
2 0 2 2
1 1 4 5
*/
```

```go=1
func main() {
    for i := 0; i < 5; i++ {
        defer fmt.Println(i)
    }
}
/*
4
3
2
1
0
*/
```
## 使用情境
- 打開文件後, 關閉/釋放文件
- 接收請求後, 回覆請求
- 加鎖後, 解鎖

### 釋放資源
```go=1
func fileSize(filename string) int64 {
    // 根據文件名稱打開
    f, err := os.Open(filename)
    
    if err != nil {
    // 錯誤回傳, 觸發defer
     return 0
    }
    // 延遲調用Close(), 這時候還不會立刻被呼叫
    defer f.Close()
    
    // 獲取文件訊息
    info, err := f.Stat()

    if err != nil {
        // 錯誤回傳, 觸發defer
        return 0
    }
    // 獲取文件大小
    size := info.Size()
    // 回傳, 觸發defer
    return size
}
```

### 加鎖解鎖
```go=1
var (
    valueByKey = make(map[string]int)
    valueByKeyGuard sync.Mutex
)

func readValue(key string) int {
    valueByKeyGuard.Lock()
    
    // 延遲解鎖
    defer valueByKeyGuard.Unlocok()
    
    return valueByKey[key]
}
```

## 誤用defer

### defer 去執行nil
```go
func main() {
	var run func() = nil

	defer run()

	fmt.Println("ithome")
}
/*
ithome
panic: runtime error: invalid memory address or nil pointer dereference
*/
```

### for loop中使用
```go
func main() {
    for {
        row, err := db.Query("select 1")
        if err != nil {
            fmt.Println(err)
        }
        defer row.Close()
    }
}
```
這種用法會在main這方法內, 一直累加很多個defer...
直到崩潰.

#### 解法, 直接再開一個匿名函數, 就會在這匿名函數結束前執行defer
```go
func main() {
    for {
        func() {
            row, err := db.Query("select 1")
            if err != nil {
                fmt.Println(err)
            }
            defer row.Close()
        }()
       
    }
}
```

### panic恢復
透過defer將匿名函數延遲執行,
panic觸發時, protectRun()函數就會結束, defer就會被觸發.
透過defer內的recoever捕捉到panic與其內容.
判斷是否是運行時的錯誤, 還是手動拋出的錯誤, 並作不同處置.

```go=1
package main

import (
	"fmt"
	"runtime"
)

type panicContext struct {
	function string
}

func protectRun(entry func()) {
	defer func() {
		if err := recover(); err != nil {
			switch err.(type) {
			case runtime.Error:
				fmt.Println("runtime: ", err)
			default:
				fmt.Println("error : ", err)
			}

		}
	}()

	entry()
}

func main() {
	fmt.Println("執行前")

	protectRun(func() {
		fmt.Println("手動觸發panic前")
		panic(&panicContext{"手動觸發!"})
		fmt.Println("手動觸發panic後")
	})

	protectRun(func() {
		fmt.Println("賦值當機前")
		var a *int
		*a = 1
		fmt.Println("賦值當機後")
	})

	fmt.Println("執行後")
}
/*
執行前
手動觸發panic前
error :  &{手動觸發!}
賦值當機前
runtime:  runtime error: invalid memory address or nil pointer dereference
執行後
*/
```

分享這個是因為...
未來很多真正使用上都會需要defer跟錯誤處理.

> [躲避 Go 1.13 defer 性能提升的姿势](https://zhuanlan.zhihu.com/p/81857521?utm_source=jp.naver.line.android&utm_medium=social&utm_oi=1116288166998482944)

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10217900)