---
title: Go_Gin
date: 2020-12-28 00:19:59
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](https://i.imgur.com/2sRi1ig.png)
來杯琴酒(Gin)+萊姆=琴蕾(Gimlet)吧(誤)

# [Gin](https://github.com/gin-gonic/gin)
Gin是一個基於Golang實做的框架, 特色是簡單!!!
- 設計精巧好懂的router/middleware系統
- 簡單好用的上下文gin.Context
- JSON、XML、DataBiding、Validation...
<!-- more -->

## 安裝Gin
```bash
go get -u github.com/gin-gonic/gin
```

## Hello It Home
```go=1
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
	router.GET("/hello", func(c *gin.Context) {
		c.Data(200, "text/plain", []byte("Hello, It Home!"))
	})
	
	router.Run()
}
```
![](https://i.imgur.com/UdN8zyi.png)

昨天的hello路由, 這樣就寫完了.
且啟動時跟收到請求時, Gin有預設的log middleware會列印出耗費時間.
且還支援RESTful API的動詞(GET/POST/PATCH/DELETE/PUT...)
這個gin.Default()作用等同於net/http包內的DefaultServeMux, 只是這是gin包裝過的預設路由引擎.
```go
// Default returns an Engine instance with the Logger and Recovery middleware already attached.
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```
這預設引擎使用了Logger()和Recovery()
Logger就是負責我們終端機上看到的log.
Recovery負責的是當有panic發生時, 就進行http status 500的錯誤處理(避免服務因此就終止了). 

## 測試 
為了方便測試, 我們把路由處理放到一個單獨的資料夾內.
又開一個test資料夾.
![](https://i.imgur.com/mE8ghFA.png)

然後安裝一下斷言包[testify](https://github.com/stretchr/testify)

`main.go`
```go=1
package main

import (
	"github.com/tedmax100/gin-angular/router"
)

func main() {
	router := router.SetupRouter()
	router.Run()
}
```

`helloRouter.go`
```go=1
package router

import (
	"github.com/gin-gonic/gin"
)

func SetupRouter() *gin.Engine {
	router := gin.Default()
	router.GET("/hello", func(c *gin.Context) {
		c.Data(200, "text/plain", []byte("Hello, It Home!"))
	})
	return router
}
```

我們會想測試這/hello會回給我們200的狀態跟Hello, It Home!
net/http包當中還提供了一樣神器[httptest](https://golang.org/pkg/net/http/httptest/)

**httptest**能讓我們快速的建立一個server, 或者建立一個recorder來紀錄response.
server我們就用Gin.
所以這裡就用recorder來捕捉response的內容來測試.

```go=1
package test

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/tedmax100/gin-angular/router"
)

func TestIHelloGetRouter(t *testing.T) {
	router := router.SetupRouter()

	w := httptest.NewRecorder()
	req, _ := http.NewRequest(http.MethodGet, "/hello", nil)

	router.ServeHTTP(w, req)

	assert.Equal(t, http.StatusOK, w.Code)
	assert.Equal(t, "Hello, It Home!", w.Body.String())
}
```
```bash
go test ./...
----------------
?       github.com/tedmax100/gin-angular        [no test files]
?       github.com/tedmax100/gin-angular/router [no test files]
ok      github.com/tedmax100/gin-angular/test   0.004s
```
測試成功!
來看看寫的測試程式碼.

net/http包也是能發出Request請求, 所以這裡就是透過http.NewRequest來對我們想測試的API發出請求.

重點在於router.ServeHTTP(w, req)

```go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```
我們的router是一個已經註冊好/hello的router了.
當收到請求後, gin會從連線池中取得一個空的context, 而不是每次都去生成一個新的context, 這樣效率會快很多.
然後再過engine.handleHTTPRequest(c), 來處理這context.

昨天提到的net/http的DefaultServeMux也有ServeHTTP(), 
只是它沒有context池子跟context上下文物件的概念來處理請求.

然後就能測試status code跟body內容了.


## URL Parameter 路徑參數
我們在寫RESTful API時, 因為都是以資源為維度在操作.
所以URL裡會有地方表示資源代碼或是名稱.
也不可能是hard code寫死. 所以這裡要透過Param()來取得這部份的表示.

helloRouter.go
```go
package router

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func SetupRouter() *gin.Engine {
	router := gin.Default()
	router.GET("/hello", func(ctx *gin.Context) {
		ctx.Data(200, "text/plain", []byte("Hello, It Home!"))
	})

	router.DELETE("/hello/:id", func(ctx *gin.Context) {
		id := ctx.Param("id")
		ctx.String(http.StatusOK, "hello DELETE %s", id)
	})
	return router
}

```
Gin就是能這樣快速的加入一個RESTful的路由.
我們的DELETE("/hello/:id"), 這裡會需要對hello id是?的資源作刪除的動作.
並且在response body加入被刪除的id.
![](https://i.imgur.com/RmMMs55.png)

來看看Context.Param()
```go
// Param returns the value of the URL param.
// It is a shortcut for c.Params.ByName(key)
//     router.GET("/user/:id", func(c *gin.Context) {
//         // a GET request to /user/john
//         id := c.Param("id") // id == "john"
//     })
func (c *Context) Param(key string) string {
	return c.Params.ByName(key)
}


type Params []Param

type Param struct {
	Key   string
	Value string
}

func (ps Params) ByName(name string) (va string) {
	va, _ = ps.Get(name)
	return
}

func (ps Params) Get(name string) (string, bool) {
	for _, entry := range ps {
		if entry.Key == name {
			return entry.Value, true
		}
	}
	return "", false
}
```
就很簡單的在Param[]裡面, 嘗試找看看有沒有這名稱.

多了個方法, 又能來寫測試了.
因為我自己還不熟TDD這樣的開發習慣, 所以我都是先寫可執行程式後, 再補單元測試.

## 重構
### 程式碼搬家去
```go
func SetupRouter() *gin.Engine {
	router := gin.Default()
	router.GET("/hello", func(ctx *gin.Context) {
		ctx.Data(200, "text/plain", []byte("Hello, It Home!"))
	})

	router.DELETE("/hello/:id", func(ctx *gin.Context) {
		id := ctx.Param("id")
		ctx.String(http.StatusOK, "hello DELETE %s", id)
	})
	return router
}
```
原本的SetupRouter這方法, 裡面有出現處理邏輯的部份.
這樣會讓這方法出現除了只定義路由與處理方法之外的職責.
我們把它們搬家.
![](https://i.imgur.com/k8USfva.png)
建立了一個handler資料夾, 並在裡面建立了一個`helloHandler.go`
```go=1
package handler

import (
	"net/http"
	"github.com/gin-gonic/gin"
)

func GetHello(ctx *gin.Context) {
	ctx.Data(200, "text/plain", []byte("Hello, It Home!"))
}

func DeleteHello(ctx *gin.Context) {
	id := ctx.Param("id")
	ctx.String(http.StatusOK, "hello DELETE %s", id)
}
```
搬家之後的SetupRouter()就變得很清爽了, 程式碼只有路由跟處理方法.
這算是設計原則中的[單一職責](https://ithelp.ithome.com.tw/articles/10191955)的應用.
```go
package router

import (
	"github.com/gin-gonic/gin"
	"github.com/tedmax100/gin-angular/handler"
)

func SetupRouter() *gin.Engine {
	router := gin.Default()
	router.GET("/hello", handler.GetHello)

	router.DELETE("/hello/:id", handler.DeleteHello)
	return router
}
```

因為搬了家, 之前寫好的測試這時一定要是ok! 跑看看測試.
```bash
go test ./...
----------------
?       github.com/tedmax100/gin-angular        [no test files]
?       github.com/tedmax100/gin-angular/router [no test files]
ok      github.com/tedmax100/gin-angular/test   0.004s
```

### 新增Delete測試
```go
func TestIHelloDeleteRouter(t *testing.T) {
	id := "123"
	router := router.SetupRouter()

	w := httptest.NewRecorder()
	req, _ := http.NewRequest(http.MethodDelete, "/hello/"+id, nil)

	router.ServeHTTP(w, req)

	assert.Equal(t, http.StatusOK, w.Code)
	assert.Equal(t, "hello DELETE "+id, w.Body.String())
}
```
列印更多資訊, 只要加上-v (verbose的縮寫)
```bash
go test -v ./...  
----------------
?       github.com/tedmax100/gin-angular        [no test files]
?       github.com/tedmax100/gin-angular/handler        [no test files]
?       github.com/tedmax100/gin-angular/router [no test files]
=== RUN   TestIHelloGetRouter
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /hello                    --> github.com/tedmax100/gin-angular/handler.GetHello (3 handlers)
[GIN-debug] DELETE /hello/:id                --> github.com/tedmax100/gin-angular/handler.DeleteHello (3 handlers)
[GIN] 2019/09/29 - 00:02:31 | 200 |       4.699µs |                 | GET      /hello
--- PASS: TestIHelloGetRouter (0.00s)
=== RUN   TestIHelloDeleteRouter
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /hello                    --> github.com/tedmax100/gin-angular/handler.GetHello (3 handlers)
[GIN-debug] DELETE /hello/:id                --> github.com/tedmax100/gin-angular/handler.DeleteHello (3 handlers)
[GIN] 2019/09/29 - 00:02:31 | 200 |       2.938µs |                 | DELETE   /hello/123
--- PASS: TestIHelloDeleteRouter (0.00s)
PASS
ok      github.com/tedmax100/gin-angular/test   0.004s
```
可以看得出來2個測試都成功.
我們舊有的測試沒問題, 新增的測試也成功.
這就是[**回歸測試Regression Testing**](https://zh.wikipedia.org/wiki/%E5%9B%9E%E5%BD%92%E6%B5%8B%E8%AF%95)


## 路由分類分組
我們剛剛的routing都是/hello開頭相關的.
現在業務變多了, 加上user相關的!

把helloRouter.go 改名成SetupRouter.go
並且透過Group(), 來建立路由分組.

```go
// Group creates a new router group. You should add all the routes that have common middlewares or the same path prefix.
// For example, all the routes that use a common middleware for authorization could be grouped.
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
	return &RouterGroup{
		Handlers: group.combineHandlers(handlers),
		basePath: group.calculateAbsolutePath(relativePath),
		engine:   group.engine,
	}
}
```

```go
package router

import (
	"github.com/gin-gonic/gin"
	"github.com/tedmax100/gin-angular/handler"
)

func SetupRouter() *gin.Engine {
	router := gin.Default()
	helloRouting := router.Group("/hello")
	{
		helloRouting.GET("", handler.GetHello)

		helloRouting.DELETE("/:id", handler.DeleteHello)
	}

	userRouting := router.Group("/user")
	{
		userRouting.GET("", handler.GetUser)

	}
	return router
}
```

userHandler.go
```go=1
package handler

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func GetUser(ctx *gin.Context) {
	uid := ctx.Param("uid")
	ctx.JSON(http.StatusOK, gin.H{
		"userId": uid,
	})
}
```

老樣子先跑原本的測試. 都ok!

透過Postman打看看
![](https://i.imgur.com/tUI0p3l.png)


再來新增測試
`userRouter_test.go`
```go=1
package test

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/tedmax100/gin-angular/router"
)

func TestIUserGetRouter(t *testing.T) {
	type User struct {
		UserId string `json:"userId"`
	}
	user := User{
		UserId: "123",
	}
	expectedBody, _ := json.Marshal(user)
	router := router.SetupRouter()

	w := httptest.NewRecorder()
	req, _ := http.NewRequest(http.MethodGet, "/user/"+user.UserId, nil)

	router.ServeHTTP(w, req)

	assert.Equal(t, http.StatusOK, w.Code)
	assert.Equal(t, string(expectedBody), w.Body.String())
}
```
```bash
go test -v ./...
--------------
?       github.com/tedmax100/gin-angular        [no test files]
?       github.com/tedmax100/gin-angular/handler        [no test files]
?       github.com/tedmax100/gin-angular/router [no test files]
=== RUN   TestIHelloGetRouter
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /hello                    --> github.com/tedmax100/gin-angular/handler.GetHello (3 handlers)
[GIN-debug] DELETE /hello/:id                --> github.com/tedmax100/gin-angular/handler.DeleteHello (3 handlers)
[GIN-debug] GET    /user/:uid                --> github.com/tedmax100/gin-angular/handler.GetUser (3 handlers)
[GIN] 2019/09/29 - 00:49:09 | 200 |       9.512µs |                 | GET      /hello
--- PASS: TestIHelloGetRouter (0.00s)
=== RUN   TestIHelloDeleteRouter
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /hello                    --> github.com/tedmax100/gin-angular/handler.GetHello (3 handlers)
[GIN-debug] DELETE /hello/:id                --> github.com/tedmax100/gin-angular/handler.DeleteHello (3 handlers)
[GIN-debug] GET    /user/:uid                --> github.com/tedmax100/gin-angular/handler.GetUser (3 handlers)
[GIN] 2019/09/29 - 00:49:09 | 200 |        3.06µs |                 | DELETE   /hello/123
--- PASS: TestIHelloDeleteRouter (0.00s)
=== RUN   TestIUserGetRouter
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /hello                    --> github.com/tedmax100/gin-angular/handler.GetHello (3 handlers)
[GIN-debug] DELETE /hello/:id                --> github.com/tedmax100/gin-angular/handler.DeleteHello (3 handlers)
[GIN-debug] GET    /user/:uid                --> github.com/tedmax100/gin-angular/handler.GetUser (3 handlers)
[GIN] 2019/09/29 - 00:49:09 | 200 |        8.62µs |                 | GET      /user/123
--- PASS: TestIUserGetRouter (0.00s)
PASS
ok      github.com/tedmax100/gin-angular/test   0.004s
```
![](https://i.imgur.com/1npXan0.png)
全綠燈~爽!!
![](https://i.imgur.com/VyBKlTl.png)


以下是我在NodeJS的Express框架設定路由分組跟用Jest+Supertest寫Api測試.
可以發現Gin+httptest+testify, 讓我可以把以前的習慣帶過來.
![](https://i.imgur.com/TFd1GKG.png)
![](https://i.imgur.com/8L3p71L.png)
我自己會儘可能補上必要的測試情境. 
因為這是能替未來的自己在這專案上省時間的解法之一.
未來回頭重構或者是交接給別人, 我也能從測試這裡開始講解就好.
不必痛苦的一開始就看業務邏輯.

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10222303)