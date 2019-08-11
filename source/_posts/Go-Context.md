---
title: Go-Context
date: 2019-07-28 22:42:02
categories: "Go"
tags:
    - Go
---
# Context
允許傳遞"Context"在goroutine之中, 手動/超時來中止routine樹等操作.
讓所有基於該context或其衍生的子context都會收到通知, 就能進行結束操作, 最後釋放goroutine. 優雅的解決goroutine啟動之後難以控制的問題.

常見的有timeout、deadline 或 只是停止工作.

## Context Interface
```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)

	Done() <-chan struct{}

	Err() error

	Value(key interface{}) interface{}
}
```
* Deadline
    獲取設置好的截止時間 ; 第二個bool返回值表示有沒有設置截止時間 
* Done
    返回一個 readonly channel, 如果該channel可以被讀取, 表示parent context 發起了cancel請求, 就能透過Done方法收到訊號後, 作結束操作.
* Err
    返回取消的錯誤原因, 為什麼context被取消
* Value
    獲得該Context上綁定的值, 是一組KV pair, 該值通常是thread safe的

## 建立Context
```go
// 通常使用context.Background()作為樹的root, 該方法只會返回一個空的context
// 就是接收請求用
// 不可cancel, 沒有設置deadline 和帶任何value的context
ctx  := context.Background()
```
```go
// 如果不需要一個全局的context, 可以用TODO一樣只會返回一個空的context
// 就是接收請求用
ctx  := context.TODO()
```
### 建立sub context
這四個With方法, 都要接收一個parent context參數.
能理解成sub context對parent context的繼承; 反過來說就是基於parent context的衍生.
這樣層層下去就能創建一個context tree, 每個節點都能有任意個sub node, 層級也能有任意多個.
```go
// 透過這樣的方式建立一個可被取消的sub context, 然後當作參數傳給goroutine使用
// func WithValue(parent Context, key, val interface{}) Context
ctx := context.WithValue(context.Background(), key, "test")
```
```go
// func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
ctx, calcel := context.WithCancel(context.Background())
```
```go
// 跟WithCancel很像, 只是多個截止時間, 表示時間到了會自動取消context; 
// 但也能手動cancel
// func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(2 * time.Second))
```
```go
// 開始執行後多少時間自動取消context
// func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
ctx, cancel := context.WithTimeout(context.Background(), 2 * time.Second)
```

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx, "goroutine1")
	go watch(ctx, "goroutine2")
	go watch(ctx, "goroutine3")

	time.Sleep(3 * time.Second)
	fmt.Println("notify stop goroutines by the context")
	cancel()

	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name, "finish the goroutine")
			return
		default:
			fmt.Println(name, "goroutine working...")
			time.Sleep(1 * time.Second)
		}
	}
}
/*
goroutine1 goroutine working...
goroutine2 goroutine working...
goroutine3 goroutine working...
goroutine1 goroutine working...
goroutine3 goroutine working...
goroutine2 goroutine working...
goroutine2 goroutine working...
goroutine1 goroutine working...
goroutine3 goroutine working...
notify stop goroutines by the context
goroutine2 finish the goroutine
goroutine3 finish the goroutine
goroutine1 finish the goroutine
*/
```
