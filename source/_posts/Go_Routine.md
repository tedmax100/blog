---
title: Goroutine 讓你用少少的線程, 能接受更多的工作, 但沒說會作比較快
date: 2020-12-21 21:22:12
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](https://i.imgur.com/cGyNZXY.jpg)
​<!-- more -->
# Goroutine
開發運行時總是會需要處理併發任務.
併發是指同一時間可以執行多個任務.
併發通常包含多執行緒, 多進程, 分佈式程序等.
Go提供的是處理多份工作的能力, 透過goroutine來進行系統調度, 把一個方法創建為goroutine時, Go會將其視為一個獨立的工作單元, 這個單元會被調度到可用的邏輯處理器上執行;
一個goroutine大小大概2kb-4kb, 非常的小, 所以要管理個上千上萬個goroutine是相對於其他語言, 比較不佔記憶體的.
Go的併發同步模型是採用CSP(Communicating Sequential Process), 是一種訊息傳遞模式, 不是透過對資料加上lock來做同步存取, 而是透過CSP在goroutine之間傳遞訊息, channel在多個goroutine之間進行同步通信與交換.
![](https://i.imgur.com/RdTf4Ni.png)

OS調度器會決定哪個threads要進行調度到CPU上執行.
Go是把單個邏輯CPU綁定到單個OS thread上, 這些邏輯CPU會來執行所有被創建的goroutine, 運行時再把每個可用的實際CPU分配一個邏輯CPU. 

![](https://i.imgur.com/zdAXAKW.png)
只有單核的情形下, Goroutine跟 NodeJS的EventLoop(1:m映射)很相似.
都是同時間只有一件事情在主線程內處理, 只是快速的在事件間切換, 但NodeJS就是單線程+IO多路複用, 特別適合IO密集型的情境.
只是處理網路請求, Node是都會吃下請求的, 但如果進行後面的處理, 就會是同步的了.
要善用全部核心, 就要依賴PM2這類的起多個Node實例.

但只要有多核心多線程, goroutine會生成多個邏輯處理器在調度器間處理, 每個上會有很多goroutines (n:m映射). 並且是透過channel交換訊息, 確保同時間只有一個goroutine可以訪問資料,  並不是透過跨線程的共享內存空間來交換, 這需要很多額外的lock來處理同步.

![](https://i.imgur.com/DjnodJq.png)


# Concurrency is not Parallelism
## Concurrency 
![](https://i.imgur.com/al6A3va.png)
一個收銀機服務一排隊列, 只是代表能同時管理很多事情, 這些事情可能只做一半就被暫停去作別的事情; 能滿足Go跟Node的設計哲學, "用較少的資源作更多的事情".

## Parallelism
![](https://i.imgur.com/0nnWfBg.png)
數個收銀機同時服務多排隊列

如上面所說的, 只要有多個P(logical CPU), 調度器會把goroutine分配到每個P上. 這會讓goroutine在不同的系統線程M上執行. 
但如果要實現平行, 需要自己讓代碼執行在有多個物理cpu上.

併發的案例 : 
```go=1
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	// 使用一個邏輯處理器給調度器用
	runtime.GOMAXPROCS(1)

	// wg + 2 表示要等待2個goroutine完成
	var wg sync.WaitGroup
	wg.Add(2)

	fmt.Println("start goroutines")

	// 創建goroutine
	go func() {
		defer wg.Done()

		for count := 0; count < 3; count++ {
			for char := 'a'; char < 'a'+26; char++ {
				fmt.Printf("%c", char)
			}
		}
	}()

	// 創建另一個goroutine
	go func() {
		defer wg.Done()

		for count := 0; count < 3; count++ {
			for char := 'A'; char < 'A'+26; char++ {
				fmt.Printf("%c", char)
			}
		}
	}()

	fmt.Println("waiting to finish")
	wg.Wait()

	fmt.Println("\nfinish Program")
}
// Output :
// tart goroutines
// waiting to finish
// ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyz
// finish Program
```
能看到2個goroutine一個接一個併發執行.

把邏輯處理器改成2
```go
runtime.GOMAXPROCS(2)
// Output :
// ABCDEFGHabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZ
```
```go=1
package main

import (
	"fmt"
	"runtime"
	"sync"
)

var wg sync.WaitGroup

func main() {
	// 使用一個邏輯處理器給調度器用
	runtime.GOMAXPROCS(1)

	// wg + 2 表示要等待2個goroutine完成

	wg.Add(2)

	fmt.Println("start goroutines")

	// 創建goroutine
	go printPrime("A")
	go printPrime("B")

	fmt.Println("waiting to finish")
	wg.Wait()

	fmt.Println("\nfinish Program")
}

func printPrime(prefix string) {
	defer wg.Done()

next:
	for outer := 2000; outer < 50000; outer++ {
		for inner := 2; inner < outer; inner++ {
			if outer%inner == 0 {
				continue next
			}
		}
		fmt.Printf("%s : %d\n", prefix, outer)
	}
	fmt.Println("completed", prefix)
}
// Output :
// tart goroutines
// waiting to finish
// B : 49801
//  B : 49807
// B : 49811
// A : 49411
// A : 49417
// A : 49429
// finish Program
```
因為查找質數會耗費不少時間, 能看到調度器有機會在第一個goroutine找到所有質數之前, 切換到另一個goroutine.



## 控制併發
> runtime.Gosched(), 告訴調度器將切換到另一個goroutine.
### 全局共享變數
```go=1
package main

import (
	"fmt"
	"runtime"
	"sync"
)

var (
	counter int
	wg      sync.WaitGroup
)

func main() {
	/* 	runtime.GOMAXPROCS(2) */
	wg.Add(2)

	go incCounter(1)
	go incCounter(2)

	wg.Wait()

	fmt.Println("finish counter:", counter)
}

func incCounter(id int) {
	defer wg.Done()

	for count := 0; count < 2; count++ {
		value := counter
		runtime.Gosched()
		value++

		counter = value
	}
}

// finish counter: 2
```
因為在存取counter這全局變數時, 沒有上鎖, 導致發生race condition在counter這變數的存取上.

Go有指令能幫助偵測race condition
```bash
go build -race
./app
```
```
==================
WARNING: DATA RACE
Read at 0x0000005ef648 by goroutine 7:
  main.incCounter()
  main.go:30 +0x6f

Previous write at 0x0000005ef648 by goroutine 6:
  main.incCounter()
  main.go:34 +0x90

Goroutine 7 (running) created at:
  main.main()
  main.go:19 +0x89

Goroutine 6 (finished) created at:
  main.main()
  main.go:18 +0x68
==================
finish counter: 4
Found 1 data race(s)
```
明確的說出在第幾行會有競爭問題.

#### Lock
* atomic 原子方法(透過底層的加鎖機制, 所以還是有鎖)
```go=1
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

var (
	counter int64
	wg      sync.WaitGroup
	mutex   sync.Mutex
)

func main() {
	wg.Add(2)

	go incCounter(1)
	go incCounter(2)

	wg.Wait()
	fmt.Printf("finish:%d\n", counter)
}

func incCounter(id int) {
	defer wg.Done()

	for count := 0; count < 2; count++ {
		counter = atomic.AddInt64(&counter, 1)
	}
}

```
* sync.mutex 互斥鎖
    * 利用mutex創建一個critical section, 保證同一時間只會有一個goroutine進入執行
```go=1
package main

import (
	"fmt"
	"runtime"
	"sync"
)

var (
    // shared variable
	counter int64
	wg      sync.WaitGroup
	// mutex lock
    mutex   sync.Mutex
)

func main() {
	wg.Add(2)

	go incCounter(1)
	go incCounter(2)

	wg.Wait()
	fmt.Printf("finish:%d\n", counter)
}

func incCounter(id int) {
	defer wg.Done()

	for count := 0; count < 2; count++ {
        // lock the critical section
		mutex.Lock()
		{
			value := counter
			runtime.Gosched()

			value++
			counter = value
		}
        // unlock
		mutex.Unlock()
	}
}
// finish : 4
```

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10218483)