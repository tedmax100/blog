---
title: Docker Multi Stage Build
date: 2019-08-08 00:06:18
categories: "Docker"
tags:
    - Docker
    - Go
---
![](/images/Go/4b2d10e2b8128b3c5d8fb38235e08b4793f150f8.jpeg)
Docker 17.05版的發布了Multi-stage build, 讓image肥大的問題有了優雅的解法.
<!--more--> 
# Our Go program :
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello world!")
}
```

# Single-stage build
Dockerfile :
```dockerfile=
FROM golang:alpine
WORKDIR /app
ADD . /app
RUN cd /app && go build -o goapp
ENTRYPOINT ./goapp
```

```bash
# build image
docker build -t main .
```
```bash
docker images | grep main
```
![](https://i.imgur.com/iMNta0E.png)
![](https://i.imgur.com/2V3QnYD.png)
**352MB**的鏡像大小, 這對於要快速佈署是相當的肥大的. 
因為Go只需要編譯完成的binary檔, Go image其實只是輔助編譯source code用的.
所以透過Multi-Stage build 來減少程式的鏡像檔大小.

# Multi-Stage Build
適用在需要編譯環境的應用上(GO, C, JAVA...)
至少都會需要兩個環境的Docker image:
* 編譯環境鏡像
    * 完整的編譯引擎, 依賴庫等等
* 運行環境鏡像
    * 編譯好的二進制檔, 用來執行app, 因為沒有編譯環境, 所以體機會小上很多
使用multi-stage build, 可以使用單一的dockerfile, 降低維護複雜度.

```dockerfile=
# build stage
FROM golang:alpine AS build-env
ADD . /src
RUN cd /src && go build -o goapp

#final stage
FROM alpine
WORKDIR /app
COPY --from=build-env /src/goapp /app/
ENTRYPOINT ./goapp
```

![](https://i.imgur.com/a3B55jj.png)
**7.58MB**

# More Examples
### Import "time"
* main.go
```go
package main

import (
	"fmt"
	"time"
)

func main() {

	location, err := time.LoadLocation("Europe/Berlin")
	if err != nil {
		fmt.Println(err)
	}

	t := time.Now().In(location)

	fmt.Println("Time in Berlin:", t.Format("02.01.2006 15:04"))
}
```
build 完之後執行會出錯
![](https://i.imgur.com/IJr0Xxb.png)

搜尋該錯誤 panic: time: missing Location in call to Time.In
搜尋Google後得知, 原來時區位置是從本地文件讀取出的.
可以透過安裝tzdata, 在/usr/share/zoneinfo產生各時區的資訊; 或者複製機器上的
修改Dockefile

```dockerfile=
# build stage
FROM golang:alpine AS build-env
ADD . /src
WORKDIR /src
RUN go build -o goapp

#final stage
FROM alpine
WORKDIR /app
# RUN apk add --no-cache tzdata
# COPY --from=build-env /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build-env /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=build-env /src/goapp /app/
ENTRYPOINT ./goapp
```

### Go Module
App Code + go.mod + go.sum
```go
// main.go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run()
}
```

### 1. Build Golang App
準備官方的Golang image, 並且取別名為builder-env, 方便在之後的階段來使用
設定工作目錄, 
因為我是用go module作套件依賴管理,
這裡把路徑設定成跟我們開發環境中一樣, 都是GOPATH下(go/src)的路徑.
複製代碼, 並且安裝依賴, 編譯go app
```dockerfile=
FROM golang  AS build-env
WORKDIR /go/src/github.com/tedmax100/docker-multistage-build-demo
COPY . .
ENV GO111MODULE=on
RUN CGO_ENABLED=0 GOOS=linux go build -o main
```
### 2. Deployment image
使用scratch 來作基礎image
把編譯好的程式放在裡面;
scratch大小 比alpine還小.
如果app 需要SSL/TLS來進行訪問, 則需要複製ca-certificates
```dockerfile=
FROM scratch
WORKDIR /bin
# COPY --from=build-env /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
COPY --from=build-env /go/src/github.com/tedmax100/mahjong .
CMD ["./main"]
```
僅產生出15.Mb的image
![](https://i.imgur.com/KFvPRRt.png)

### docker build
透過--rm 刪除中間過程產生的容器
```bash
docker build --rm -t main .
```
### docker run
```bash=
docker run --rm -d -p 8081:8080 main
```
![](https://i.imgur.com/HVJsI0E.png)

[source code](https://gitlab.com/tedmax100/demo)