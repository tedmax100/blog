---
title: 3兄弟-Ring
date: 2020-12-20 22:37:57
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](https://i.imgur.com/kiOhD1T.png)
這隻又跑出來了XD
Ring其實就是雙向環鏈(circular doubled linked list)
![](https://i.imgur.com/g9YwBjN.png)
用這圖, 是想表達, 我們有一個歌單
可以單向依序放到完,  當然也能選擇循環依序播放阿 !!!
Ring可以滿足這行為的操作!!
​<!-- more -->
# [Ring](https://golang.org/pkg/container/ring/#Ring.Link)
只有一個Value屬性,開發者可以任意操作.
prev, next都是給操作方法操作用的.
```go
type Ring struct {
    next, prev *Ring
    Value      interface{} 
}
```
​
## Ring vs List
可以發現Ring的結構跟[List](https://ithelp.ithome.com.tw/articles/10214704)超像.
- 結構差別是List是由List和Element類別兩個聯合表示; 而Ring自己就能代表值和關聯.
- Ring一開始要指定初始元素個數, 被創建出來後, 長度就不可變; 但List則不必, 也沒這必要.
- 通過var聲明的Ring的零值是長度為1的環鏈; 而List的零值則是長度為0的雙向鏈結, 因為只有root ptr, 並沒有指向任何元素.
- Ring的Len()是O(N), 它需要把每個元素走訪一次直到走到自己; List的則是O(1), 因為有個len變數在紀錄長度. 這在某些情境上, 大大影響效能.
​
## 初始化
1. 透過New(size), 生出長度為size的Ring
```go
// New creates a ring of n elements.
func New(n int) *Ring {
    if n <= 0 {
        return nil
    }
    r := new(Ring)
    p := r
    for i := 1; i < n; i++ {
        p.next = &Ring{prev: p}
        p = p.next
    }
    p.next = r
    r.prev = p
    return r
}
​
func (r *Ring) init() *Ring {
    r.next = r
    r.prev = r
    return r
}
```
2. 透過宣告, 生出長度為1的Ring
```go
var ring變數 ring.Ring
```
​
## 操作方法
- 透過next取得下一個元素
```go
func (r *Ring) Next() *Ring {...}
```
- 透過prev取得前一個元素
```go
func (r *Ring) Prev() *Ring {...}
```
- 讓目前環鏈依據目前所在的元素位置, 往前(n<0)或是往後(n>0)移動數個位置
```go
func (r *Ring) Move(n int) *Ring {...}
```
- 讓目前的環鏈與另一個環鏈作連結, s會插入到r目前指向的元素後面, 返回插入前, r.Next()所表示的元素
如果r跟s指向的是同一個ring, 就會刪掉r跟s之間的元素, 
被刪掉的元素會組成一個新的ring, 返回的就是指向這新ring的指針
```go
func (r *Ring) Link(s *Ring) *Ring {...}
```
- 從當前環鏈所在的Next()依序刪除n個元素, 返回值是被刪除的元素們
```go
func (r *Ring) Unlink(n int) *Ring {...}
```
- 取得環鏈元素個數, 複雜度為O(N)
```go
func (r *Ring) Len() int {...}
```
- 傳入一個函數, Do會依序地讓每個元素的Value當作參數去執行該函數; 類似JS的map()
也能透過累加數值在外部變數上
或者實做[策略模式](https://zh.wikipedia.org/wiki/%E7%AD%96%E7%95%A5%E6%A8%A1%E5%BC%8F), 執行每個元素的封裝行為.
但要避免函數f去改變了r, 會發生不可預期的行為.
```go
func (r *Ring) Do(f func(interface{})) {...}
```
## 基本範例 
```go=1
package main
​
import (
    "container/ring"
    "fmt"
)
​
// 宣告一個要給Do()執行的函數, 用來列印值而已
var printRing = func(v interface{}) {
    fmt.Print(v.(int), "->")
}
​
// 只是用來呼叫r.Do跟代入printRing, 只是多一個換行
func PrintRing(r *ring.Ring) {
    r.Do(printRing)
    fmt.Println()
}
​
func main() {
    // 透過var 來宣告ring
    var varRing ring.Ring
    // 查看透過var宣告的ring的長度
    fmt.Println("查看透過var宣告的ring的長度: ", varRing.Len())
    fmt.Println("----------------------")
    //透過New創建10個元素的ring
    r := ring.New(10)
    // 查看透過New()初始化ring的長度
    fmt.Println("查看透過New()初始化ring的長度: ", r.Len())
    // 給ring中每個元素進行走訪並且給值
    for i := 0; i < 10; i++ {
        r.Value = i
        // 取得下一個元素
        r = r.Next()
    }
    fmt.Print("r : ")
    PrintRing(r)
​
    // 往後移動ring的指向
    r = r.Move(2)
    fmt.Println("ring 向後移動2個位置的元素值:", r.Value)
​
    // 往前移動ring的指向
    r = r.Move(-8)
    fmt.Println("ring 向前移動8個位置的元素值:", r.Value)
​
    // 從ring當前指向開始刪除n個元素
    deletedElm := r.Unlink(2)
    fmt.Print("r 所剩下的元素 : ")
    PrintRing(r)
    fmt.Print("從r刪除的元素 : ")
    PrintRing(deletedElm)
​
    // 準備第2個ring r2
    r2 := ring.New(3)
    for i := 0; i < 3; i++ {
        r2.Value = i + 10
        r2 = r2.Next()
    }
    fmt.Print("r2 : ")
    PrintRing(r2)
​
    fmt.Println("現在r的指向在 :", r.Value)
    // Link r 跟 r2
    fmt.Print("Link r 跟 r2 : ")
    linkedRing := r.Link(r2)
    PrintRing(r)
​
    // 以原本r.Next()開始走訪
    fmt.Print("以原本r.Next()開始走訪 : ")
    PrintRing(linkedRing)
}
/*
查看透過var宣告的ring的長度:  1
----------------------
查看透過New()初始化ring的長度:  10
r : 0->1->2->3->4->5->6->7->8->9->
ring 向後移動2個位置的元素值: 2
ring 向前移動8個位置的元素值: 4
r 所剩下的元素 : 4->7->8->9->0->1->2->3->
從r刪除的元素 : 5->6->
r2 : 10->11->12->
現在r的指向在 : 4
Link r 跟 r2 : 4->10->11->12->7->8->9->0->1->2->3->
以原本r.Next()開始走訪 : 7->8->9->0->1->2->3->4->10->11->12->
*/
```
## 輪播範例
```go=1
package main
​
import (
    "container/ring"
    "fmt"
    "time"
)
​
// song類別
type song struct {
    name   string
    artist string
    length time.Duration
}
​
// 定義歌單
var (
    songs = []song{
        {
            name:   "Something Just Like This",
            artist: "The Chainsmokers",
            length: 247,
        },
        {
            name:   "Blame",
            artist: "Calvin Harris",
            length: 214,
        },
        {
            name:   "Wolves",
            artist: "Selena Gomez",
            length: 197,
        },
        {
            name:   "Sing You To Sleep",
            artist: "Matt Cab",
            length: 236,
        },
    }
)
​
func main() {
    // 載入歌單
    songList := ring.New(len(songs))
    repeatedCnt := 0
​
    for i := 0; i < songList.Len(); i++ {
        songList.Value = songs[i]
        songList = songList.Next()
    }
​
   // 開始播放
    for {
        if repeatedCnt == 1 {
            break
        }
        songList.Do(func(v interface{}) {
            time.Sleep((v.(song).length / 100) * time.Second) // 加速播放
            fmt.Printf("現正播放%s, 演唱者為%s\n", v.(song).name, v.(song).artist)
        })
        repeatedCnt++
        fmt.Printf("播放次數 : %d\n", repeatedCnt)
    }
​
    fmt.Println("播放完畢")
}
/*
現正播放Something Just Like This, 演唱者為The Chainsmokers
現正播放Blame, 演唱者為Calvin Harris
現正播放Wolves, 演唱者為Selena Gomez
現正播放Sing You To Sleep, 演唱者為Matt Cab
播放次數 1: 
現正播放Something Just Like This, 演唱者為The Chainsmokers
現正播放Blame, 演唱者為Calvin Harris
現正播放Wolves, 演唱者為Selena Gomez
現正播放Sing You To Sleep, 演唱者為Matt Cab
播放次數 2: 
現正播放Something Just Like This, 演唱者為The Chainsmokers
現正播放Blame, 演唱者為Calvin Harris
現正播放Wolves, 演唱者為Selena Gomez
現正播放Sing You To Sleep, 演唱者為Matt Cab
播放次數 3: 
播放完畢
*/
```
​
## 3兄弟各自適合的使用場景
1. List 
    - FIFO queue
2. Heap
    - 排序
    - Priority queue
    - 定時器
3. Ring
    - 上面提到的輪播
    - 保存近n筆操作日誌
​
應該還有更多, 我暫時還沒想到, 歡迎大家補充給我.
感謝各位.

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10214925)