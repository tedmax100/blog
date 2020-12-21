---
title: Go_Context_電話蟲
date: 2020-12-21 21:26:53
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
想像一下
* 如果用多個goroutine來處理一個請求, 那怎在這些goroutine之間共享request訊息.
* 每一個請求都應該要有個**超時限制**
* 處理超時, 設定3s後超時
    * 在函數被調用的過程中, 還剩下多久才超時?
    * 需要在哪裡存放這超時訊息
    * 怎樣在請求過程處理中,使其停止?
* 更方便的控制goroutine的關閉, 如果不想多創造channel的話.

![](https://miro.medium.com/max/1002/1*4zwrwPEwhn8_2A_m2zrc7Q.jpeg)
​<!-- more -->

# [Context](https://golang.org/pkg/context/#pkg-variables)
Context最常見的是上下文這詞來說明, 但其實應用上我們都只看上文.
叫做[*語境*](https://www.itsfun.com.tw/%E8%AA%9E%E5%A2%83/wiki-3918416-3625395)可能更貼切.
透過傳遞Context用來簡化對於處理單個請求的多個goroutine之間的資料共享、超時和退出等操作, 手動/超時等操作.
當我們在做線程切換時, 就需要保存當前的狀況, 載入下一個線程需要的stack跟資料暫存器.
這資料暫存器跟stack其實就是Context.

由於context能衍生出子context, 
所有能讓基於該context或其衍生的子context都會收到通知, 就能進行結束操作.
最後釋放goroutine. 優雅的解決goroutine啟動之後難以控制的問題.

常見的有timeout、deadline 或 只是停止工作.

Go提供了可以攜帶Value的context、可以取消的context和可以設置timeout的context.

## Context Interface
```go=1
type Context interface {
    // 獲取設置好的截止時間 ; 第二個bool返回值表示有沒有設置截止時間 
	Deadline() (deadline time.Time, ok bool)
    // 返回一個 readonly channel, 如果該channel可以被讀取, 表示parent context 發起了cancel請求, 就能透過Done方法收到訊號後, 作結束操作.
	Done() <-chan struct{}
    // 返回取消的錯誤原因, 為什麼context被取消
	Err() error
    // 讓goroutine共享資料, 透過獲得該Context上綁定的值, 是一組KV pair, 是thread safe的;
    // 不存在則返回nil
	Value(key interface{}) interface{}
}
```

## 建立root Context
```go
// 通常使用context.Background()作為樹的root, 該方法只會返回一個空的context
// 就是接收請求用
// 不可cancel, 沒有設置deadline 和帶任何value的context
ctx  := context.Background()
```
```go
// 如果在開發階段, 還不清楚是要怎麼用該context, 可以用TODO(), 
// 一樣是返回一個空的context
ctx  := context.TODO()
```
## 建立sub context
這四個With方法, 都要接收一個parent context參數.
能理解成sub context對parent context的繼承; 反過來說就是基於parent context的衍生.
這樣層層下去就能創建一個context tree, 每個節點都能有任意個sub node, 層級也能有任意多個.

記得一定要呼叫cancel(), 不然會leak.
能透過```go vet```指令來檢查有沒有leak.

### WithValue
```go
// 透過這樣的方式建立一個可被取消的sub context, 然後當作參數傳給goroutine使用
// func WithValue(parent Context, key, val interface{}) Context
ctx := context.WithValue(context.Background(), key, "test")
```
### WithCancel
```go
// func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
ctx, calcel := context.WithCancel(context.Background())
```
```go=1
package main

import (
	"context"
	"log"
	"os"
	"time"
)

var logger *log.Logger
var key string = "name"

func main() {
	logger = log.New(os.Stdout, "", log.Ltime)
	// 建立一個cancel context
	ctx, cancel := context.WithCancel(context.Background())

	// 建立數個withValue context, 繼承於ctx,  並給值
	valueCtx := context.WithValue(ctx, key, 1)
	valueCtx2 := context.WithValue(ctx, key, 2)
	go watch(valueCtx)
	go watch(valueCtx2)

	time.Sleep(4 * time.Second)

	logger.Println("任務停止")
	// 發出取消
	cancel()

	// 確保工作結束
	time.Sleep(1 * time.Second)
}

func watch(ctx context.Context) {

	for {
		select {
		case <-ctx.Done():
			//接收到取消訊號
			logger.Println("任務", ctx.Value(key), ":任務停止...")
			return
		default:
			//取出值
			var value int = ctx.Value(key).(int)
			logger.Println("任務", ctx.Value(key), ":工作中")
			time.Sleep(time.Duration(value) * time.Second)
		}
	}
}
/*
20:24:50 任務 1 :工作中
20:24:50 任務 2 :工作中
20:24:51 任務 1 :工作中
20:24:52 任務 2 :工作中
20:24:52 任務 1 :工作中
20:24:53 任務 1 :工作中
20:24:54 任務停止
20:24:54 任務 2 :任務停止...
20:24:54 任務 1 :任務停止...
*/
```


### WithDeadline
```go
// 跟WithCancel很像, 只是多個截止時間, 表示時間到了會自動取消context; 
// 傳入的不是duration而是確切時間
// 但也能手動cancel
// func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(2 * time.Second))
```
```go=1
package main

import (
	"context"
	"log"
	"os"
	"time"
)

var logger *log.Logger

func do(ctx context.Context) {
	if deadline, ok := ctx.Deadline(); ok == true {
		logger.Println("deadline: ", deadline)
	}
	for {
		select {
		case <-ctx.Done():
			// logger.Println("deadline is over")
			logger.Println(ctx.Err())
			return
		default:

			logger.Println("do")
			time.Sleep(1 * time.Second)
		}
	}

}
func main() {
	logger = log.New(os.Stdout, "", log.Ltime)

	d := time.Now().Add(2 * time.Second)
	// 現在時間的2秒後的時間就是deadline
	ctx, cancel := context.WithDeadline(context.Background(), d)

	defer cancel()
	logger.Println("start")
	go do(ctx)

	time.Sleep(3 * time.Second)
}
/*
21:20:25 start
21:20:25 deadline:  2019-09-22 21:20:27.844274236 +0800 CST m=+2.000197284
21:20:25 do
21:20:26 do
21:20:27 context deadline exceeded
*/
```
### WithTimeout
```go
// 開始執行後多少時間自動取消context, 傳入的是duration
// func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
ctx, cancel := context.WithTimeout(context.Background(), 2 * time.Second)
```
```go=1
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"
)

var logger *log.Logger

func doForever(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			logger.Println(ctx.Err())
			return
		default:
			logger.Println("doForever")
			time.Sleep(1 * time.Second)
		}
	}
}

func do1second(ctx context.Context) {
	select {
	case <-ctx.Done():
		logger.Println(ctx.Err())
		return
	default:
		time.Sleep(1 * time.Second)
		logger.Println("do1second")
	}
}

func main() {
	logger = log.New(os.Stdout, "", log.Ltime)
	// 建立一個timeout context,  3秒後沒返回就發出超時
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)

	defer cancel()
	logger.Println("start")
	go doForever(ctx)
	go do1second(ctx)

	time.Sleep(4 * time.Second)
}
/*
21:10:55 start
21:10:55 doForever
21:10:56 doForever
21:10:56 do1second
21:10:57 doForever
21:10:58 context deadline exceeded
*/
```

## Context Tree
前面提到了建立sub context, 看看上下文樹的結構
```go=1
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```
chidren這屬性用來紀錄用此context所建立出來的sub context, 
同時Context屬性是當前的context. 
```go=1
package main

import "context"

var cancelBefore = false

func main() {
	c, cCancel := context.WithCancel(context.Background())

	c1, cf1 := context.WithCancel(c)
	defer cf1()

	c2, cf2 := context.WithCancel(c)
	defer cf2()

	c11, cf11 := context.WithCancel(c1)
	defer cf11()

	c12, cf12 := context.WithCancel(c1)
	defer cf12()

	if cancelBefore {
		cCancel()
	}

	for k, c := range map[string]context.Context{`c1`: c1, `c11`: c11, `c12`: c12, `c2`: c2} {
		var s string
		if c.Err() != nil {
			s = `cancelled`
		} else {
			s = `not cancelled`
		}
		println(k + ` is ` + s)
	}

	if !cancelBefore {
		cCancel()
	}
}
```
![](https://i.imgur.com/hptA3bo.png)
每個context相互連結, 只要對C發出cancel, 所有屬於它的children context也將會被cancel.

### cancel
```go=1
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
    // 關閉done這個blocking channel
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
    // 這裡對每個children呼叫cancel
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

## 使用原則
- 不要把context放在struct成員之中, 應該要透過參數作傳遞; 但如果該struct本身也是方法的參數, 就可以.
- 變數名取為ctx, 且放在參數列的第一個, 返回也是.
- 在傳遞context時, 不要傳遞nil, 不然在trace追蹤時會斷鏈, 此時可以傳遞TODO()
- Context是thread safe的, 能放心的在各個goroutine之間傳遞
- 可以把一個context實例, 傳遞給任意數量的goroutine. context被cancel()時, 所有的goroutine都會接收到取消訊號.


## 使用情境1 : 全鍊路追蹤
透過WithValue在請求的根埋入一組數據, key是生成好的TracId(用戶id).
SpanId表示處理該trace的服務代碼, ParentId表示呼叫方的SpanId.
透過這樣子的方式就能在http的接口端, 埋入對應資訊.
彙整時, 只要對TraceId撈取, 對ParentId做排序, 就能得到一條完整的調用鏈紀錄.

## 使用情境2 : 對於耗時任務作主動性的取消, 即時的釋放資源
最常見的就是使用time.After在select等待接收到資訊, 作任務的返回.
```go
func Task() {
  select {
    case <- time.After(2*time.Second):
        return
  }
}
```
如果使用WithTimeout、WithDeadline、WithCancel
就能把這取消的權力, 反轉過來變成是在調用方了.
有沒有一種依賴反轉(IOC)的feel? 然後ctx作為參數用外部傳入(DI).

還有許多使用情境, 之後的範例應該會很常用到, 像是資料庫的慢查詢.

> [Go Concurrency Patterns: Context](https://blog.golang.org/context)

> [Go Context 官方範例](https://golang.org/src/context/example_test.go)

> [Go vet](https://golang.org/cmd/vet/)

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10219405)