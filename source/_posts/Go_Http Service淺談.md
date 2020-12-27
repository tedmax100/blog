---
title: Go_Http Service淺談
date: 2020-12-28 00:18:37
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
現在幾乎什麼服務都是走Http協議, 提供WebAPI給client使用.
NodeJS幾年前盛起, 一小部份原因也是他做WebAPI很好寫沒太多複雜的設定.
Go在建立http服務也是頗簡單. 
2者也都內建web server ; 以前寫C#還得放到IIS, Java則是放到Tomcat....

Go提供[net/http包](https://golang.org/pkg/net/http/), 能提供路由、靜態文件、Template、Cookie、檔案系統等.
​<!-- more -->

來建立個會回Hello It Home的http服務吧.
```go=1
package main

import (
	"io"
	"log"
	"net/http"
)

func main() {
	// 設置路由跟處理方法
	http.HandleFunc("/", HelloHandler)
	// 設置監聽的port
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func HelloHandler(w http.ResponseWriter, req *http.Request) {
	io.WriteString(w, "Hello, It Home!\n")
}
```

啟動Http服務器
```bash
go run main.go
```
打開瀏覽器輸入localhost:8080
![](https://i.imgur.com/QbPpdfG.png)

![](https://i.imgur.com/DUCUxCz.png)

這樣就啟動一個最基本的Http服務了.
行數少少的, 就能起一個服務了.

## Http Handler 
第一個參數是路由匹配的字串.
第二個參數是func(ResponseWriter, *Request)這類型的方法.
該方法第一個參數是負責Respons, 另一個則是http請求.
```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}
```
可以看到內部程式就1行, 執行DefaultServeMux.HandleFunc
來看看什麼是DefaultServeMux

## DefaultServeMux
默認的路由集合.
```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
	h       Handler
	pattern string
}

type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
muxEntry 儲存路由路徑和對應的routing handler.
因為Handler是一個接口, 所以只要滿足的方法都能作註冊.
![](https://i.imgur.com/MR103Yl.png)

有兩個全局方法能針對DefaultServeMux作設定
第一個參數是告訴它要監聽哪個TCP地址.
第二個參數是能帶入自定義的server mux; 
但我們傳nil, 就會使用DefaultServeMux, 來當預設的多工器.
certFile是我們如果要建立https的服務時, 要給它憑證證書的公鑰、私鑰.
```go
func ListenAndServe(addr string, handler Handler) errorerror
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error
```

## Request
一個請求最常見的有Method、Header、Body、Cookie...
### Post handler
```go=1
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"net/http"
)

func main() {
	// 設置路由跟處理方法
	http.HandleFunc("/", HelloHandler)
	http.HandleFunc("/post", PostHandler)
	// http.HandleFunc("/get", GetHandler)
	// 設置監聽的port
	log.Fatal(http.ListenAndServe(":8080", nil))
}

type ReqData struct {
	Method  string
	Body    string
	Headers map[string][]string
	Cookie  []*http.Cookie
	Params  map[string][]string
	Url     string
}

func (r ReqData) String() string {
	b, err := json.Marshal(r)
	if err != nil {
		return err.Error()
	}
	return string(b)
}

func PostHandler(w http.ResponseWriter, req *http.Request) {
	log.Println("req method: ", req.Method)

	if req.Method == "POST" {
		body, err := ioutil.ReadAll(req.Body)
		if err != nil {
			http.Error(w, "read request body error", http.StatusInternalServerError)
		}
		reqdata := ReqData{
			Method:  req.Method,
			Body:    string(body),
			Headers: req.Header,
			Params:  req.URL.Query(),
			Cookie:  req.Cookies(),
			Url:     req.URL.String(),
		}

		fmt.Fprint(w, reqdata.String())
		return
	}
	http.Error(w, "invalid request method", http.StatusMethodNotAllowed)
}

func GetHandler(w http.ResponseWriter, req *http.Request) {
	log.Println("req method: ", req.Method)

	io.WriteString(w, "Hello, It Home!\n")
}

func HelloHandler(w http.ResponseWriter, req *http.Request) {
	io.WriteString(w, "Hello, It Home!\n")
}
```
這裡我把幾個常用到的request屬性的型別給響應在response而已.
Body則是一個接口類型, 其內容是個[]byte, 可以透過讀取這[]byte來獲得內容.
```go
ReadCloser interface {
    Reader
    Closer
}
```
可以看到Header跟Query string的Params其實內部實現都是map.
這裡最後我是把reqdata的字串寫入到w這ResponseWriter.
mux會把這w根據response的content-type作回應.
但我這裡沒表明用什麼type, 就只是一般的text.

透過Postman發出POST請求, 就會得到回應了.
![](https://i.imgur.com/0dcOtcR.png)
太丑了XD, 來美化一下.

## Response
response跟request結構大體接近. 也有header、body、cookie
為了讓回傳是Json, 我加上response.header是"Content-Type:application/json", 告訴瀏覽器這回應的是json格式.

```go=1
func (r ReqData) Marshal() []byte {
	b, err := json.Marshal(r)
	if err != nil {
		return []byte(err.Error())
	}
	return b
}

func PostHandler(w http.ResponseWriter, req *http.Request) {
	log.Println("req method: ", req.Method)

	if req.Method == "POST" {
		body, err := ioutil.ReadAll(req.Body)
		if err != nil {
			http.Error(w, "read request body error", http.StatusInternalServerError)
		}
		reqdata := ReqData{
			Method:  req.Method,
			Body:    string(body),
			Headers: req.Header,
			Params:  req.URL.Query(),
			Cookie:  req.Cookies(),
			Url:     req.URL.String(),
		}
		w.Header().Set("Content-Type", "application/json")
		w.Write(reqdata.Marshal())
		return
	}
	http.Error(w, "invalid request method", http.StatusMethodNotAllowed)
}
```
![](https://i.imgur.com/XX0kZHb.png)
![](https://i.imgur.com/iwsg56w.png)

### Redirect
只要Header標明是302, 然後header給上Location跟轉址的url.
就能輕鬆完成.
```go=1
w.Header().Set("Location", "https://google.com")
w.WriteHeader(302)
```

## Http Status
net/http內很貼心的把RFC有定義的status全列舉在這了.
使用時就只要呼叫http就會看到智慧選單, Status開頭且是int的都是.
```go=1
const (
	StatusContinue           = 100 // RFC 7231, 6.2.1
	StatusSwitchingProtocols = 101 // RFC 7231, 6.2.2
	StatusProcessing         = 102 // RFC 2518, 10.1

	StatusOK                   = 200 // RFC 7231, 6.3.1
	StatusCreated              = 201 // RFC 7231, 6.3.2
	StatusAccepted             = 202 // RFC 7231, 6.3.3
	StatusNonAuthoritativeInfo = 203 // RFC 7231, 6.3.4
	StatusNoContent            = 204 // RFC 7231, 6.3.5
	StatusResetContent         = 205 // RFC 7231, 6.3.6
	StatusPartialContent       = 206 // RFC 7233, 4.1
	StatusMultiStatus          = 207 // RFC 4918, 11.1
	StatusAlreadyReported      = 208 // RFC 5842, 7.1
	StatusIMUsed               = 226 // RFC 3229, 10.4.1

	StatusMultipleChoices   = 300 // RFC 7231, 6.4.1
	StatusMovedPermanently  = 301 // RFC 7231, 6.4.2
	StatusFound             = 302 // RFC 7231, 6.4.3
	StatusSeeOther          = 303 // RFC 7231, 6.4.4
	StatusNotModified       = 304 // RFC 7232, 4.1
	StatusUseProxy          = 305 // RFC 7231, 6.4.5
	_                       = 306 // RFC 7231, 6.4.6 (Unused)
	StatusTemporaryRedirect = 307 // RFC 7231, 6.4.7
	StatusPermanentRedirect = 308 // RFC 7538, 3

	StatusBadRequest                   = 400 // RFC 7231, 6.5.1
	StatusUnauthorized                 = 401 // RFC 7235, 3.1
	StatusPaymentRequired              = 402 // RFC 7231, 6.5.2
	StatusForbidden                    = 403 // RFC 7231, 6.5.3
	StatusNotFound                     = 404 // RFC 7231, 6.5.4
	StatusMethodNotAllowed             = 405 // RFC 7231, 6.5.5
	StatusNotAcceptable                = 406 // RFC 7231, 6.5.6
	StatusProxyAuthRequired            = 407 // RFC 7235, 3.2
	StatusRequestTimeout               = 408 // RFC 7231, 6.5.7
	StatusConflict                     = 409 // RFC 7231, 6.5.8
	StatusGone                         = 410 // RFC 7231, 6.5.9
	StatusLengthRequired               = 411 // RFC 7231, 6.5.10
	StatusPreconditionFailed           = 412 // RFC 7232, 4.2
	StatusRequestEntityTooLarge        = 413 // RFC 7231, 6.5.11
	StatusRequestURITooLong            = 414 // RFC 7231, 6.5.12
	StatusUnsupportedMediaType         = 415 // RFC 7231, 6.5.13
	StatusRequestedRangeNotSatisfiable = 416 // RFC 7233, 4.4
	StatusExpectationFailed            = 417 // RFC 7231, 6.5.14
	StatusTeapot                       = 418 // RFC 7168, 2.3.3
	StatusMisdirectedRequest           = 421 // RFC 7540, 9.1.2
	StatusUnprocessableEntity          = 422 // RFC 4918, 11.2
	StatusLocked                       = 423 // RFC 4918, 11.3
	StatusFailedDependency             = 424 // RFC 4918, 11.4
	StatusTooEarly                     = 425 // RFC 8470, 5.2.
	StatusUpgradeRequired              = 426 // RFC 7231, 6.5.15
	StatusPreconditionRequired         = 428 // RFC 6585, 3
	StatusTooManyRequests              = 429 // RFC 6585, 4
	StatusRequestHeaderFieldsTooLarge  = 431 // RFC 6585, 5
	StatusUnavailableForLegalReasons   = 451 // RFC 7725, 3

	StatusInternalServerError           = 500 // RFC 7231, 6.6.1
	StatusNotImplemented                = 501 // RFC 7231, 6.6.2
	StatusBadGateway                    = 502 // RFC 7231, 6.6.3
	StatusServiceUnavailable            = 503 // RFC 7231, 6.6.4
	StatusGatewayTimeout                = 504 // RFC 7231, 6.6.5
	StatusHTTPVersionNotSupported       = 505 // RFC 7231, 6.6.6
	StatusVariantAlsoNegotiates         = 506 // RFC 2295, 8.1
	StatusInsufficientStorage           = 507 // RFC 4918, 11.5
	StatusLoopDetected                  = 508 // RFC 5842, 7.2
	StatusNotExtended                   = 510 // RFC 2774, 7
	StatusNetworkAuthenticationRequired = 511 // RFC 6585, 6
)

```

> 這篇文章有一張net/http的流程圖能學習; 能了解服務從開始到請求過來的過程
> [Golang构建HTTP服务（一）--- net/http库源码笔记](https://www.cnblogs.com/Survivalist/p/10033169.html)

其實這樣只要路由一多, 很難維護.
有的還要寫middleware, 或是處理session等等的.
我剛學的時候是看[gorilla/mux](https://github.com/gorilla/mux)
它可以方便的定義.
```go=1
r := mux.NewRouter()
indexroute := r.PathPrefix("/").Subrouter()
indexroute.Use(MiddlewareOne)

healthchecks := r.PathPrefix("/health").Subrouter()
healthchecks.Use(MiddlewareTwo)

log.Fatal(http.ListenAndServe("localhost:8000", r))
```
但現在更火熱的是[Gin](https://github.com/gin-gonic/gin)這套Web框架.
明天來玩看看.

這套可能不是效能最優的框架.
But...很多時候效能瓶頸可能都不在這些框架上;
而是我們對Go或者是Sql甚至是架構上的處理跟了解不夠深入.
所以我都是挑生態圈活躍的框架來深入.
畢竟很多雷, 大家都幫忙踩完解掉了^^

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10221840)