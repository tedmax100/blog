---
title: Go_Channel
date: 2020-12-21 21:23:32
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](https://i.imgur.com/iQ4hriX.jpg)
# Channel
channel能夠在多個goroutine之間作數據交換, 任何時間, 同時只能有一個goroutine來存取通道進行發送或獲取資料. Channel就像是一個輸送帶, 遵守著FIFO的規則, 保證收發資料的順序.
![](https://i.imgur.com/qQx12ku.png)
​<!-- more -->
通道就像是在捷運等公共場所很多人的情況下, 大家在遵守著排隊的習慣, 目的是避免擁擠、插隊導致的低效資源使用與交換過程.
多個goroutine為了搶奪存取資料, 勢必造成執行效率的低下, 使用queue是一種高效率的同步存取方式, channel就是一種queue一樣的結構.

- var 通道名稱 chan 通道類型
- chan的空值是nil
- 聲明完通道後, 要透過make來產生實例
- 通道實例  := make(chan 通類類型,  [bufferSize int]), 晚點會講解有沒有加上buffer size的使用差別
- 發送資料 <- ,   通道變數 <- 值    
- 接收資料 
    - 阻塞式接收  資料變數 := <- 通道變數
    - 非阻塞式接收  資料變數, ok := <- 通道變數
        - ok : 表示是否收到資料
    - 接收後忽略   <- 通道變數
        - 執行到這句會變成阻塞, 直到收到資料, 但收到的資料會被忽略
        - 最常用在goroutine間阻塞式地收發實現併發同步.
    - 用for range 進行多個資料的接收
    -  相較於阻塞式, 會造成較高的CPU佔用. 
    -  如果需要超時檢測, 可配合select和計時器channel使用.

Ex 1 : 無緩衝通道的併發列印, 發布者與訂閱者的簡易範例
```go=1
package main

import (
	"fmt"
)

func printer(c chan int) {

	// 無限循環等待資料
	for {
		// 從channel 取得資料
		data := <-c

		if data == 0 {
			fmt.Println("break")
			break
		}
		fmt.Println(data)
	}

	// 通知main 已經結束了
	c <- 0
}

func main() {
	// 建立一個int channel
	c := make(chan int)

	// 把channel 傳入, 讓它開始等待資料餵入
	go printer(c)

	for i := 1; i <= 10; i++ {
		// 餵入資料給channel
		c <- i
	}

	// 通知printer 結束 ; 這裡 0 表示結束
	c <- 0

	// 等printer 結束通知
	<-c
}
```

Ex2 : 單向通道, 只能發送或是接收
- 只能發送, var 通道變數 chan <- 類型 
- 只能接收, var 通道變數 <- chan 類型
```go
ch := make(chan int)

var sendOnlyCh chan <- int = ch
var recvOnlyCh <- chan int = chkjj
```

先來看一下內建的Timer的原始碼, 會發現他的屬性C也是個只能接收資料的通道.
透過從通道C獲得, 就能得知定時器到期這個事件的到來.
只要時間倒數一到, 定時器會對自己發送一個time.Time類型的值.

```go
// The Timer type represents a single event.
// When the Timer expires, the current time will be sent on C,
// unless the Timer was created by AfterFunc.
// A Timer must be created with NewTimer or AfterFunc.
type Timer struct {
	C <-chan Time
	r runtimeTimer
}
```
```go=1
package main

import (
	"fmt"
	"time"
)

func main() {
	// 設置每2秒就觸發的定時器
	timer := time.NewTimer(time.Second * 2)

	defer timer.Stop()

	for {
		// 從channel取值
		fmt.Println(<-timer.C)
		// 重新設置每一秒就觸發的定時器
		timer.Reset(time.Second)
	}
}
```

上面宣告通道時都沒帶上最後一個參數
這參數定義的是緩衝空間的大小.

剛剛我們用的叫做**無緩衝區的通道**, 這種通道類型, 就是沒有宣告buffer size的通道.
先來補充上面講的unbuffered 跟buffered channel的差異.

**Unbuffered Channel 無緩衝區的通道**
![](https://i.imgur.com/rmhLiOP.png)
無緩衝通道沒有任何緩衝區容量, 所以需要兩個goroutine(1發1收)準備好進行資料互換.
當發布者goroutine嘗試把資料發送到unbuffered channel時, 訂閱者goroutine等待接收資料的話.
該channel會把接著要發送資料的goroutine給lock作等待, 直到有其他訂閱者goroutine嘗試接收走.
如果有訂閱者goroutine嘗試從unbuffered接收資料,  但也沒有另一個發布者goroutine來發送資料的話, 該訂閱者goroutine也會被lock作等待.

圖中的3、4、5, 就是兩方嘗試作交換的動作.
```go=1
package main

import (
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup

func main() {
	// unbuffered channel
	baton := make(chan int)

	wg.Add(1)

	go Runner(baton)

	// start from 1
	baton <- 1

	wg.Wait()
}

func Runner(baton chan int) {
	var newRunner int
	// get baton from channel
	runner := <-baton

	fmt.Printf("Runner %d Running With Baton\n", runner)

	if runner != 4 {
		newRunner = runner + 1
		fmt.Printf("Runner %d To The Line\n", newRunner)
		// 創建另一個goroutine, 等有發布者把接力棒丟進去通道內
		go Runner(baton)
	}

	time.Sleep(100 * time.Millisecond)

	if runner == 4 {
		fmt.Printf("Runner %d Finished, Race Over\n", runner)
		wg.Done()
		return
	}

	fmt.Printf("Runner %d Exchange With Runner %d\n", runner, newRunner)
	baton <- newRunner
}
/*
Runner 1 Running With Baton
Runner 2 To The Line
Runner 1 Exchange With Runner 2
Runner 2 Running With Baton
Runner 3 To The Line
Runner 2 Exchange With Runner 3
Runner 3 Running With Baton
Runner 4 To The Line
Runner 3 Exchange With Runner 4
Runner 4 Running With Baton
Runner 4 Finished, Race Over
*/
```

**Buffered Channel 有緩衝區的通道**
上面所提的unbuffered channel可以視為是size為0的buffered channel.
![](https://i.imgur.com/oCOB31x.png)
有緩衝區的通道, 具有buffer size, 所以發跟收兩方能單獨作業.
可是當buffer已滿或是空的, 就跟unbuffered一樣的變成同步行為了.

> **為什麼要限制長度而不是提供無限長度的通道呢?**
> channel是在兩個goroutine之間通信的橋樑.
> 因此必然有一方提供資料, 一方作為消費者接收資料.
> 當供給速度遠大過接收的處理速度時, 如果通道不限制長度, 則記憶體會不斷膨脹, 直到app崩潰.
> 因此發送資料量必須在消費方處理量+通道長度的範圍內, 才能正確的處理.

結論 :
對於buffered channel 長度為C, 
則通道中第K個接收完成操作發生在第K+C個發送完成之前.
如果把C設成0則對應unbuffered channel, 也就是第K個接收完成在K+1個發送完成之前.
因為該類型只能同步發送一個.

故可以根據channel的buffer size來控制goroutine的最大數量.
**不要透過共享變數+Mutex來進行操作, 應該透過channel來共享**

## Channel的循環接收
```go
// 通道內的資料可以透過for range進行多個資料的接收操作, 一次for就得到一筆資料
for data := range channel {

}
```
```go=1
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int)

	go func() {
		for i := 3; i >= 0; i-- {
			ch <- i
			time.Sleep(time.Second)
		}
	}()

	for data := range ch {
		fmt.Println(data)
		if data == 0 {
			break
		}
	}
}
/*
3
2
1
0
*/
```

### Channel的關閉回收
channel是一個reference object, 和map類似.
只要沒有外部在引用就會被回收掉. 但也能夠主動的關閉.
[探究golang的channel和map内存释放问题](http://xiaorui.cc/2018/10/19/%E6%8E%A2%E7%A9%B6golang%E7%9A%84channel%E5%92%8Cmap%E5%86%85%E5%AD%98%E9%87%8A%E6%94%BE%E9%97%AE%E9%A2%98/)

透過 **close(通道變數)**
被關閉的channel一樣可以被訪問,只是會觸發**panic**

### 發送資料給被關閉的channel
```go=1
package main

import "fmt"

func main() {
	ch := make(chan int)

	close(ch)

	fmt.Println("ptr: %p, len: %d\n", ch, len(ch))

	ch <- 1
}
/*
ptr: %p, len: %d
 0xc000076060 0
panic: send on closed channel
*/
```
被關閉的channel, 其實不會是nil, 但如果嘗試發送資料給被關閉的通道, 
就會發出panic.

### 從被關閉的channel接收資料
```go=1
package main

import "fmt"

func main() {
	ch := make(chan string, 2)

	ch <- "0"
	ch <- "1"

	close(ch)

	for i := 0; i < cap(ch)+1; i++ {
		v, ok := <-ch
		fmt.Println(v, ok)
	}
}

/*
0 true
1 true
   false
*/
```
我們在執行for loop之前就關閉了通道, 但裡面的資料不會被釋放, 通道也不會消失.
我們還是可以從被關閉的channel取回資料來處理的, 然後通道這時停止阻塞.
前兩個結果表示, 還是可以進行接收資料的動作的.
這是字串通道, 第三行的  false, 表示通道在關閉狀態下取出的值. v表示該類型的默認值, 因為是字串類型, 所以返回空字串, false表示沒有獲取成功, 因為通道已經空了.

## 使用Select
用來響應多個channel的操作, 行為類似switch case, 只是每個case被一個channel操作取代了.
在每個case都會對應一個channel的收發過程.
當收發完成後, 會出發case中對應的語句.
多個操作在每次select中挑選一個進行回應.
不過如果select中至少兩個以上的case同時被滿足觸發, 就只會**隨機**挑一個case執行.
```go
select (
    case 成功操作ch1 :
        響應操作1
    case 成功操作ch2 :
        響應操作2
    default: 
        其他case都沒有滿足觸發時, 會執行默認case, 避免select被阻塞.
)
```
|操作|語句範例|
|:-:|:-:|
|接收任意資料|case <- ch:|
|接收資料到變數上|case d := <- ch|
|發送資料|case ch <- 100 ;|

### Deadlock
#### 範例1 沒有default case:
使用select但沒有default case, 上面提到這預設是為了不被阻塞用
也沒有發布者對channel發送資料, 導致main這goroutine被阻塞導致deadlock.
只要channel的另一方有goroutine會發送資料, 那怕是幾天才發一筆, 都不會造成deadlock, 頂多是block.
所以訂閱者執行時, 會檢查channel另一邊有沒有發布者已經註冊了, 不然就是拋deadlock panic.
但接收者先送資料,  沒人收卻不會拋panic的, 最多是channel滿了被block.

如果channel是nil channel也是一樣的, 因為也會無法從中讀取資料, 也是會造成阻塞操作.
```go
package main

func main() {
	dataCh := make(chan int, 5)

	select {
	    case <-dataCh:
	}
}
/*
fatal error: all goroutines are asleep - deadlock!
goroutine 1 [chan receive]:
*/
```

```go=1
package main

import "log"

func main() {
	dataCh := make(chan int, 5)

	select {
	case <-dataCh:
    // 補上default case來避免阻塞
	default:
		log.Println("default case executed")
	}
}
/*
2019/09/21 16:56:18 default case executed
*/
```

#### 範例2  沒有半個case :
一樣一直阻塞導致deadlock
```go=1
package main

func main() {
	select {}
}
/*
fatal error: all goroutines are asleep - deadlock!
*/
```

### 隨機挑滿足的case
看個例子滿足多個case下, 會隨機挑一個滿足的case執行對應操作.
```go
package main

import (
	"log"
)

func main() {
	dataCh := make(chan int, 5)
	go func() {
		for i := 0; i < 5; i++ {
			select {
			case dataCh <- 1:
				log.Println("send 1")
			case dataCh <- 2:
				log.Println("send 2")
			case dataCh <- 3:
				log.Println("send 3")
			}
			if i == 4 {
				close(dataCh)
			}
		}
	}()

	for i := 0; i < 5; i++ {
		log.Printf("receive %v\n", <-dataCh)
	}
}
/*
2019/09/21 16:32:32 send 1
2019/09/21 16:32:32 send 1
2019/09/21 16:32:32 send 3
2019/09/21 16:32:32 send 3
2019/09/21 16:32:32 send 2
2019/09/21 16:32:32 receive 1
2019/09/21 16:32:32 receive 1
2019/09/21 16:32:32 receive 3
2019/09/21 16:32:32 receive 3
2019/09/21 16:32:32 receive 2
*/
```

### break跳脫
思考一下, for 裡面包select , 在case內break
```go=1
package main

import (
	"fmt"
	"time"
)

func test() {
	i := 0
	for {
		select {
		case <-time.After(time.Millisecond * time.Duration(500)):
			i++
			if i == 3 {
				fmt.Println("break now")
				break
			}
			fmt.Println("inside the select: ")
		}
		fmt.Println("inside the for: ")
	}
}

func main() {
	test()
}
/*
inside the select: 
inside the for: 
inside the select: 
inside the for: 
break now
inside the select: 
inside the for:  
...
*/
```
break在這種使用下, 是無法跳出for之外.
只能使用標籤, 搭配break或是goto離開.

```go=1
func test() {
	i := 0
	END:
	for {
		select {
		case <-time.After(time.Millisecond * time.Duration(500)):
			i++
			if i == 3 {
				fmt.Println("break now")
				break END
			}
			fmt.Println("inside the select: ")
		}
		fmt.Println("inside the for: ")
	}
}
```
```go=1
func test() {
	i := 0
	for {
		select {
		case <-time.After(time.Millisecond * time.Duration(500)):
			i++
			if i == 3 {
				fmt.Println("break now")
				goto END
			}
			fmt.Println("inside the select: ")
		}
		fmt.Println("inside the for: ")
	}
	END:
}

```

## 對channel的操作行為整理
|操作|nil channel|closed channel|not-closed & not nil channel|
|:-:|:-:|:-:|:-:|
|close|panic|panic|success close|
|ch<-|block|panic|block or sucess write|
|<-ch|block|read zero value|block or read success|


看得出來對channel不熟的話, 很容易panic.
尤其是在close操作上.
來整理一下怎樣的關閉通道, 能全身而退, 安全的在各goroutine之間結束.

### [The Channel Closing Principle](https://go101.org/article/channel-closing.html)

1. 別再訂閱方這裡關閉channel
2. 如果有多個發布者對上同一個channel, 這情況下, 也別在發布端這裡作關閉
3. 不要去關閉一個已經被關閉的channel
4. 不要送資料去一個已經被關閉的channel

那我們在發布端跟訂閱端這裡的使用場景就可分成
- 一個發布者, 多個訂閱者
- 多個發布者, 一個訂閱者
- M個發布者, N個訂閱者

### 一個發布者, 多個訂閱者
![](https://i.imgur.com/S9i9Prz.png)

```go=1
package main
package main

import (
	"log"
	"math/rand"
	"sync"
	"time"
)

// 一個發布者, 多個訂閱者
// 因為只有一個發布者對上channel, 所以由發布者自己決定什麼時候關閉通道
func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	// 隨機數字的最大值
	const Max = 100000
	// 訂閱者數量
	const NumSubscribers = 100

	wgSubscribers := sync.WaitGroup{}
	wgSubscribers.Add(NumSubscribers)

	// 資料通道
	dataCh := make(chan int)

	// 發布者
	go func() {
		for {
			// 當剛好出現0時
			if value := rand.Intn(Max); value == 0 {
				// 唯一的發布者可自己關閉通道
				close(dataCh)
				return
			} else {
				dataCh <- value
			}
		}
	}()

	//  訂閱者
	for i := 0; i < NumSubscribers; i++ {
		go func() {
			defer wgSubscribers.Done()

			//一直從channel接收資料直到通道關閉, 且都沒資料為止
			for value := range dataCh {
				log.Println(value)
			}
		}()
	}

	wgSubscribers.Wait()
}
```

### 多個發布者, 一個訂閱者
![](https://i.imgur.com/C1DTcjh.png)
```go=1
package main

import (
	"log"
	"math/rand"
	"sync"
	"time"
)

func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	const Max = 100000
	// 發布者數量
	const NumPublishers = 1000

	wgSubscriber := sync.WaitGroup{}
	wgSubscriber.Add(1)
	// 資料通道
	dataCh := make(chan int)
	// 停止訊號通道, 發訊號給他的是訂閱者, 訂閱者因為自己不能關閉通道, 會違反原則
	// 發布者收到停止訊號後, 就會停止發布並且返回
	stopCh := make(chan struct{})

	// 創建多個發布者
	for i := 0; i < NumPublishers; i++ {
		go func() {
			for {
    // 如果只有一個select 內有從stopCh取值跟送值給dataCh這兩個case.
    // 當同時兩個條件都滿足下, 是會發生隨機挑一個case去執行的無法預估的情況.
    // 所以第一個select只會有從stopCh取值作提早返回和default case避免阻塞用.
				select {
				// 發布者對資料通道是發布者的角色
				// 但是對停止訊號通道則是訂閱者的角色
				case <-stopCh:
					return
				default:
				}

				select {
				case <-stopCh:
					return
				case dataCh <- rand.Intn(Max):
				}
			}
		}()
	}

	//  訂閱者
	go func() {
		defer wgSubscriber.Done()

		for value := range dataCh {
			if value == Max-1 {
				// 訂閱者對停止事件通道的角色則是發布的作用,
				// 所以由他負責關閉沒有違反原則, 且也只有他一位.
				close(stopCh)
				return
			}

			log.Println(value)
		}
	}()

	wgSubscriber.Wait()
}
```
### M個發布者, N個訂閱者
![](https://i.imgur.com/RATwkQF.png)
最複雜的case
```go=1
package main

import (
	"log"
	"math/rand"
	"strconv"
	"sync"
	"time"
)

// 不能讓發布者或是訂閱者來關閉資料通道, 且不能讓發布者這邊來關閉額外的訊息通道來通知其他所有角色退出.
// 引入主持人這角色在這情境下, 來關閉訊息通道
func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	const Max = 100000
	// 訂閱者數量
	const NumSubscribers = 10
	// 發布者數量
	const NumPublishers = 1000

	wgSubscribers := sync.WaitGroup{}
	wgSubscribers.Add(NumSubscribers)

	// 資料通道
	dataCh := make(chan int)
	// 停止訊號通道, 給仲裁角色用來發送訊號的
	stopCh := make(chan struct{})

	// 一個長度為1 的通道, 主要是用來告訴主持人說該關閉通道了
	// 看是發送者發起還是接收者發起的
	toStop := make(chan string, 1)

	var stoppedBy string

	// 主持人, 就block自己, 直到從toStop取值成功, 再來關閉訊息通道
	go func() {
		stoppedBy = <-toStop
		close(stopCh)
	}()

	// 創建多個發布者
	for i := 0; i < NumPublishers; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
				// 某一個發布者決定停止, 發訊號過去給主持人
					select {
					case toStop <- "sender#" + id:
					default:
					}
					return
				}

				// 嘗試從停止通道中取值, 或者不阻塞往下繼續執行
				select {
				case <-stopCh:
					return
				default:
				}

				// 嘗試從停止通道中取值, 或者發送資料到資料通道
				select {
				case <-stopCh:
					return
				case dataCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	// 創建多個訂閱者
	for i := 0; i < NumSubscribers; i++ {
		go func(id string) {
			defer wgSubscribers.Done()

			for {
				// 嘗試從停止通道中取值, 或者不阻塞往下繼續執行
				select {
				case <-stopCh:
					return
				default:
				}

				// 嘗試從停止通道中取值, 或者從資料通道取值
				select {
				case <-stopCh:
					return
				case value := <-dataCh:
					if value == Max-1 {
						select {
						case toStop <- "receiver#" + id:
						default:
						}
						return
					}

					log.Println(value)
				}
			}
		}(strconv.Itoa(i))
	}

	wgSubscribers.Wait()
	log.Println("stopped by", stoppedBy)
}
```
![](https://i.imgur.com/0QSEw23.png)

**好像資料通道跟主持人專用通道, 都沒人去負責Close() ??**
前面提過
因為只要大家都沒在用該通道, 不論是否有沒有主動去close().
最終該通道就會被GC掉, 因為沒人在引用該通道了.


### [Pub-Sub == 觀察者模式 ?](https://hackernoon.com/observer-vs-pub-sub-pattern-50d3b27f838c)
Pub-Sub中間都會有個第三個組件message broker或者event bus/channel, 負責作調度跟管理.
觀察者則是直接由主題變化時, 通知所有觀察者.
所以這裡有channel的例子其實都是Pub-Sub.

Pub-Sub
![](https://hackernoon.com/hn-images/1*-GHFC93E4ODwNc98IE5_vA.gif)

觀察者
![](https://hackernoon.com/hn-images/1*s1kclXywIwae86iNa7cKZQ.png)

接著會陸續介紹幾種併發模型跟Context

ps:
別任意地無限建立goroutine 並且裡面有這樣寫法, 還沒任何的return
```go
for {
  xxxx
}
```
這會導致CPU被莫名其妙吃光, 因為CPU Time都花費在for(1) loop上了.
Channel本身可以是非阻塞操作讓出CPU時間, 但for (1) loop不會
[參考來源](https://stackoverflow.com/questions/57717635/golang-for-select-blows-up-cpu)

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10218923)