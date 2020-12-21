---
title: Interface & OOP  就說你是鴨子! 你就是要呱呱叫
date: 2020-12-20 22:47:46
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](/images/Go/download.jpeg)
​<!-- more -->
# Interface
一個interface(接口) 就是包含了一系列行為的method集合.
好處:
1. 能建立低耦合的系統
2. 透過這些被定義在接口的抽象行為, 讓要在多個單獨組件間彼此組合/通信會變得更為容易.
3. 隱藏每個Class對其實現的細節
4. Reusability, 因為可重複利用, 能把一些複雜問題給簡化.

## Go Interface
Go沒有真正的繼承, 所以沒有OOP那種該類別實際告訴大家我實現了某個接口這種聲明;
所以對於實現Interface是透過隱性的向上轉型的方式(Duck typing), 在程式代碼的上下文判定struct是否實現了接口聲明的方法.
所以只要該類型實現了該接口所有方法就是實現了該接口.

example :
```go=1
package main

import (
	"fmt"
)

type Engine interface {
	Start()
	Stop()
}

// CarEngine並沒繼承Engine也沒宣告自己實現了Engine
type CarEngine struct {
}
// CarEngine有自己的公開方法Start()
func (c CarEngine) Start() {
	fmt.Println("Car engine is started")
}

func (c CarEngine) Stop() {
	fmt.Println("Car engine is stoped")
}

type TrainEngine struct {
}

func (t TrainEngine) Start() {
	fmt.Println("Train engine is started")
}

func (t TrainEngine) Stop() {
	fmt.Println("Train engine is stoped")
}
// Starting和Stoping 的參數要求代入的是Engine這類型
func Starting(e Engine) {
	e.Start()
}

func Stoping(e Engine) {
	e.Stop()
}

func main() {
	carEngine := CarEngine{}
	trainEngine := TrainEngine{}
    // 這裡會檢查CarEngine和TrainEngine是否有實現Engine的全部方法
	engines := []Engine{
		carEngine, trainEngine,
	}

	for _, engine := range engines {
		Starting(engine)
		Stoping(engine)
	}
}
// 如果把TrainEngine的Stop刪除
// 在48行就會在編譯時期被檢查出錯誤
```
因為Duck typing幾乎都出現在動態語言上,  程式寫起來飛快,但錯誤往往都是在執行時才能被發現. 靜態語言就是能在編譯時期發現這類的錯誤.


Go採取了折衷的方法, 在安全和靈活之間取得平衡:
1. 靜態類型
2. 隱性實現
3. 只有某個類型的變數實現了某個接口的全部方法, 這個變數才能在要求使用該接口的地方.

#### 一個類型可以實現多個接口
```go=1
// io.Writer
type Writer interface {
        Write(p []byte) (n int, err error)
}
// io.Closer
type Closer interface {
        Close() error
}
```
```go=1
type Socket struct {
}

func (s *Socket) Write(p []byte) (n int, err error) {
	fmt.Println("Write has be involked")
	return 0, nil
}

func (s *Socket) Close() error {
	fmt.Println("Close has be involked")
	return nil
}

func usingWriter(writer io.Writer) {
	writer.Write(nil)
}

func usingCloser(closer io.Closer) {
	closer.Close()
}

func main() {
	s := new(Socket)
	usingWriter(s)
	usingCloser(s)
}
// Output :
// Write has be involked
// Close has be involked
```
![](https://tedmax100.github.io/images/Go/interface1.png)
#### 多個類型可以實現同樣的接口(polymorphism)
```go=1
type Service interface {
	Start()
	Log(string)
}

type Logger struct{}

func (g *Logger) Log(l string) {
	fmt.Println(l)
}

type GameService struct {
	Logger
}

func (g *GameService) Start() {
	fmt.Println("game service start")
}

func main() {
	var s Service = new(GameService)
	s.Start()
	s.Log("hello")
}
// Output :
// game service start
// hello
```
![](https://tedmax100.github.io/images/Go/interface2.png)
#### 接口的嵌套組合
```go=1
type device struct {
}

// 實現
func (d *device) Write(p []byte) (n int, err error) {
	return 0, nil
}

// 實現
func (d *device) Close() error {
	return nil
}

/*
// WriteCloser is the interface that groups the basic Write and Close methods.
type WriteCloser interface {
	Writer
	Closer
}
// Implementations must not retain p.
type Writer interface {
	Write(p []byte) (n int, err error)
}
*/
func main() {
	// 宣告io.WriteClose, 並賦予device的實例
	var wc io.WriteCloser = new(device)

	wc.Write(nil)

	wc.Close()

	// 宣告io.Writer, 並賦予device的實例
	var writeOnly io.Writer = new(device)

	writeOnly.Write(nil)
}
```
![](https://tedmax100.github.io/images/Go/interface3.png)
### 空接口 interface{}
interface{}是接口類型的特殊形式; 空接口沒有任何方法, 所以任何類型都沒必要去實現空接口; 反過來說, 任何值都滿足空接口的實現需求, 所以它可以保存任何值, 也能從空接口中取出值.
空接口類型類似C#, Java中的Object, C的void*.
空接口內部只保存了對象的類型和指針, 所以在使用上會比較慢一些.

```go=1
// eface = empty interface
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

```go=1
var any interface{}

any = 1
fmt.Println(any)

any = false
fmt.Println(any)
// Output :
// 1
// false
```

```go=1
var a int =1
var i interface{} = a 
var b int = i
// 第三行會報錯
// cannot use i (type interface{}) as type int in assigment : need type assertion
// 因為i 在此時還是interface{}類型, 並不是int類型
// 要使用類型斷言
var b int = i.(int)
```

## 接口斷言 Type Assertions
Type Assertion是對於interface value的一種操作方法.
語法格式 
```go
t, ok := i.(T)
```
i 代表實現接口的變數
T 表示轉換的目標類型
t 表示轉換後的變量
ok 檢查i接口是否實現T類型的效果

鳥和豬有不同的特性, 一個能飛能走, 一個只能走.
讓鳥跟豬各自實現Flyer和Walker的接口.
然後實例被放進interface{}的map中, interface{}表示空接口, 所以什麼類型都能放.
透過斷言操作來操作各接口.
```go=1
type Flyer interface {
	Fly()
}

type Walker interface {
	Walk()
}

type bird struct {
}

func (b *bird) Fly() {
	fmt.Println("bird can fly")
}

func (b *bird) Walk() {
	fmt.Println("bird can walk")
}

type pig struct {
}

func (p *pig) Walk() {
	fmt.Println("pig can walk")
}

func main() {
	animals := map[string]interface{}{
		"bird": new(bird),
		"pig":  new(pig),
	}

	for name, obj := range animals {
		f, isFlyer := obj.(Flyer)
		w, isWalker := obj.(Walker)

		fmt.Printf("name: %s isFlyer: %v isWalker: %v\n", name, isFlyer, isWalker)

		if isFlyer {
			f.Fly()
		}
		if isWalker {
			w.Walk()
		}
	}
}
// Output :
// name: bird; isFlyer: true, isWalker: true
// bird can fly
// bird can walk
// name: pig; isFlyer: false, isWalker: true
// pig can walk
```
上面寫法會很多if 
能用type  switch簡化
```go
switch obj := obj.(type) {
case Flyer:
    fmt.Printf("name: %s\n", name)
    obj.Fly()
case Walker:
    fmt.Printf("name: %s\n", name)
    obj.Walk()
		}
```
## Go OOP
### 封裝
透過package級別做封裝
私有成員跟方法在Go是以小寫開頭的, 只有在該package內可見.
公開成員跟方法是以大寫開頭.
```go=1
type Bag struct {
// private property for Bag
    item []int
}
// public method for Bag
func (b *Bag) Insert(itemid int) {
    b.items = append(b.items, itemid)
}

func main() {
    b := new(Bag)
    b.Insert(1002)
}
```

Go沒有建構式, 透過簡單工廠方法來實現
```go
type Bag struct {
// private property for Bag
    item []int
}

// simple factory method
func NewBag() Bag {
    return &Bag{}
}
```

### 繼承
Go其實沒有繼承, 都是依靠組合(composition), 允許嵌入組合.
也因為沒有繼承, 就不會出現可多重繼承裡會出現的[死亡鑽石問題](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem).

只要嵌入一個匿名類型的組合就等同於實現了繼承, 
如果只是嵌入struct那跟[脆弱基類](https://en.wikipedia.org/wiki/Fragile_base_class)是一樣的脆弱, 所以會透過嵌入接口, 來提早檢查問題.

### 多態
Go 依賴接口來實現這特性.
只要對象實現相同的接口, Go就能處理不同類型的那些對象.
```go=1
package main

import "fmt"

type Shape interface {
	Area() int64
}

type Rectangle struct {
	width, height int64
}

func NewRectangle(width, height int64) *Rectangle {
	return &Rectangle{
		width:  width,
		height: height,
	}
}

func (r *Rectangle) Area() int64 {
	return r.width * r.height
}

type Circle struct {
	radius int64
}

func NewCircle(radius int64) *Circle {
	return &Circle{
		radius: radius,
	}
}

func (c *Circle) Area() int64 {
	return c.radius * c.radius
}

func main() {
	r := NewRectangle(10, 5)
	c := NewCircle(5)
	s := []Shape{r, c}

	for _, shape := range s {
		fmt.Println(shape.Area())
	}
}
```

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10215623)