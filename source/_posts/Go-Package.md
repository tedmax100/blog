---
title: Go Package
date: 2019-07-14 15:31:31
categories: "Go"
tags:
    - Go
---
![](/images/Go/source-files-to-package.001.png)
[Everything you need to know about packages in Go](https://medium.com/rungo/everything-you-need-to-know-about-packages-in-go-b8bac62b74cc)

### Package(包)
程式碼的目錄, 可以重複利用程式的方案, 方便維護。
Go默認提供很多package, 像是fmt、is等。
開發者也可以創建自己的package。

package要求所有檔案的第一行添加package名稱，標示該文件所歸屬的package。
```go
package 包名稱
```
* 一個目錄下的同級檔案屬於同一個package
* package名稱可以與目錄不同名稱, 但盡可能一樣
* main package為應用程式執行的entry point; 若沒有main package則無法編譯成可執行的檔案在bin下
* package name, Go團隊建議簡單扁平為原則。 所以盡量避免下划線、中划線和參雜大寫字母。

#### Creating a package
1. 可執行包(executable package)
    可自己執行，表示有main package
2. 工具包(utility package)
    不可自己執行，但是可以給可執行包做擴展應用的作用
    
![](https://i.imgur.com/1gvLawQ.png)

```go
// main.go
package main

import (
	"fmt"
	. "hello/math"
)

func main() {
	fmt.Println("hello")
	fmt.Println(Average([]float64{1, 2}))
}
```
```go
// math/math.go
package math

func Average(xs []float64) float64 {
	total := float64(0)
	for _, x := range xs {
		total += x
	}
	return total / float64(len(xs))
}
```
```bash
# 編譯hello package 
cd $GOPATH/src/hello; 
go install;
# 因為有main package, 所以會安裝到$GOPATH/bin 作為可執行包
```
![](https://i.imgur.com/boE8wD0.png)

```bash
# 編譯hello package 
cd $GOPATH/src/hello/math; 
go install;
# 因為沒有main package, 所以會安裝到$GOPATH/pkg下 作為工具包
```
![](https://i.imgur.com/fhzkZRr.png)


#### Import package
使用import package，Go會先在 $GOROOT/src下尋找指定的package。
若找不到就往$GOPATH/src目錄下尋找。
找不到就會報出編譯錯誤。

```go
package main

import (
  // fmt位於$GOROOT/src下，找到!
  "fmt"
  // gin並不在$GOROOT/src, 接著找$GOPATH/src找github.com這目錄，找到往內找gin-gonic目錄，再找gin package
  "github.com/gin-gonic/gin"
  // 
  . "github.com/go-sql-driver/mysql"
)
```

#### Nested package
在一個package內嵌套令一個package; 使用上只要指名路徑關係. 
![](https://i.imgur.com/o5fWzfp.png)
![](https://i.imgur.com/JQPpOK5.png)

```go
// math/math/extend/min.go
package extend

func init() {
	fmt.Println("extend ==> init()")
}

func Min(a float64, b float64) float64 {
	if a >= b {
		return a
	}
	return b
}
```
```go
// math/math.go
package math

import (
	"fmt"
	"hello/math/extend"
)
func init() {
	fmt.Println("math ==> init()")
}

func Average(xs []float64) float64 {
	total := float64(0)
	for _, x := range xs {
		total += x
	}
	return total / float64(len(xs))
}

func Min(a float64, b float64) float64 {
	return extend.Min(a, b)
}
```
```go
// main.go
package main

import (
	"fmt"
	. "hello/math"
)

func init() {
	fmt.Println("main ==> init()")
}

func main() {
	fmt.Println("hello")
	fmt.Println(Average([]float64{1, 2}))
	fmt.Println(Min(1, 2))
}
```

### Package Initialization
![](https://i.imgur.com/e8y24gO.jpg)

#### 工廠模式自動註冊-管理多個packge
![](https://i.imgur.com/1Dn0qBn.png)

```go
// base/factory.go
package base

// define interface for Class
type Class interface {
	Do()
}

var (
	// 存放註冊好的 類別工廠資訊
	factoryByName = make(map[string]func() Class)
)

// 註冊一個類別工廠
func Register(name string, factory func() Class) {
	factoryByName[name] = factory
}

// 根據name創建對應的類別
func Create(name string) Class {
	if f, ok := factoryByName[name]; ok {
		return f()
	}
	panic("name not found")
}
```

```go
// ex1/reg.go
package ex1

import (
	"fmt"
	"github.com/tedmax100/factory/base"
)
type Class1 struct {
}

func (c *Class1) Do() {
	fmt.Println("class1")
}

func init() {
	base.Register("Class1", func() base.Class {
		return new(Class1)
	})
}
```

```go
// ex2/reg.go
package ex1

import (
	"fmt"

	"github.com/tedmax100/factory/base"
)

type Class2 struct {
}

func (c *Class2) Do() {
	fmt.Println("class2")
}

func init() {
	base.Register("Class2", func() base.Class {
		return new(Class2)
	})
}
```

```go
// main.go
package main

import (
	"github.com/tedmax100/factory/base"
	_ "github.com/tedmax100/factory/ex1"
	_ "github.com/tedmax100/factory/ex2"
)
//因為上面使用匿名導入了ex1 & ex2 package.
//main()執行前, 這兩個package的init()會被調用, 而註冊了class1 & class2
func main() {
	c1 := base.Create("Class1")
	c1.Do()

	c2 := base.Create("Class2")
	c2.Do()
}
```