---
title: Go Module & Go Vendor
date: 2019-07-14 15:32:19
categories: "Go"
tags:
    - Go
---
![](/images/Go/go-packages.jpg)
## Go Module基礎

### 出現原因
1. GOPATH不符合一般開發者的習慣; 大家習慣用maven, node module之類的方式.
2. GOPATH無法有效管理板依賴, 沒有辦法表明所依賴的包的版本.

### 環境準備
* Go version >= 1.11
* GO111MODULE=on

### GoMod effect immport package 
* 可以在$GOPATH之外的地方建立專案
* 該專案Go Module開啟後, 下載的package會放在$GOPATH/pkg/mod下.
![](https://i.imgur.com/Op2TRi6.png)
* $GOPATH/bin的功能依然保持

#### Go Mod Commands
![](https://i.imgur.com/vHzYvzT.png)
有兩種方式能定義一個正確的Go module
```bash
// 在$GOPATH/src的目錄下, 建立合理的module路徑
// 進入該module目錄, 執行下面命令
go mod init [module name]
///
```bash
// 在任意地方, 建立好module路徑
// 在該目錄下, 執行
go mod init [folder/]module name
```
就會在該專案下生出了go.mod文件了.
![](https://i.imgur.com/PIIW4Bz.png)

### Syntax of go.mod
* module 
    * 定義模組路徑
* go
    * 定義go version
* require
    * 指定依賴的功能包和其版本或是[預設是最新版]
* exclude
    * 忽略該功能包和其版本
* replace
    * 替換依賴的功能包
```
module my/package
go 1.12
require other/thing v1.0.2
require new/thing/v2 v2.3.4
exclude old/thing v1.2.3
replace bad/thing v1.4.5 => good/thing v1.4.5
```

#### Go Mod Require
* 安裝一下logrus
```bash
go get github.com/sirupsen/logrus
```
**go.mod**的內容
```
module modtest
go 1.12
require github.com/sirupsen/logrus v1.4.2 // indirect
```
此時把v1.4.2 改成v1.4.1
執行
```bash
go mod download
```
**go.mod**的內容
```
module modtest
go 1.12
require github.com/sirupsen/logrus v1.4.1 // indirect
```
也會發生$GOPATH/pkg/mod/github.com/sirupsen目錄下,多了logrus@v1.4.1和1.4.2版本的源碼
![](https://i.imgur.com/5WlkJoT.png)


#### Go Mod Exclude
**go.mod**的內容
```
module modtest
go 1.12
require github.com/sirupsen/logrus v1.4.2 // indirect

exclude github.com/gin-gonic/gin v1.4.0
```
```bash
go get github.com/gin-gonic/gin
```
會發現應該是要下載當前最新板的v1.4.0的gin; 但因為有exclude gin 1.4.0 ;
所以改成下載v1.3.9
![](https://i.imgur.com/mW1aVji.png)

**go.mod**的內容
```
module modtest
go 1.12
require (
	github.com/gin-contrib/sse v0.1.0 // indirect
	github.com/gin-gonic/gin v1.3.0 // indirect
	github.com/golang/protobuf v1.3.2 // indirect
	github.com/mattn/go-isatty v0.0.8 // indirect
	github.com/sirupsen/logrus v1.4.2
	github.com/ugorji/go v1.1.7 // indirect
	gopkg.in/go-playground/validator.v8 v8.18.2 // indirect
	gopkg.in/yaml.v2 v2.2.2 // indirect
)
exclude github.com/gin-gonic/gin v1.4.0
```
如果exclude指定gin的依賴功能包, 該功能包會避開該版號作安裝

#### Go Mod Replace
如果有package被replace, 則編譯時會使用對應的項目來作取代.
1. 與require類似, 可以指向令一個repo
2. 又或是指向本地的一個目錄

**gomodtest**
```go
// go.mod
module modtest
go 1.12
require github.com/sirupsen/logrus v1.4.2 // indirect
```
```go
// modtest.go
package gomodtest

import (
	log "github.com/sirupsen/logrus"
)

func Init() {
	log.Info("godmodtest init")
}

func Exec() {
	log.Info("godmodtest exec")
}
```

**gomaintest
```go
// go.mod
module github.com/tedmax100/gomaintest
go 1.12
replace github.com/tedmax100/modtest => ../gomodtest
```

```go
// main.go
package main

import (
	modtest "github.com/tedmax100/modtest"
)

func main() {
	modtest.Exec()
}
```
執行結果
![](https://i.imgur.com/h5OoYQS.png)  


***notes***
* Replace和Exclude都只對當前這module有影響, 對其他功能包不會去影響到 ;
其他功能包自己的replace也不會影響到這包.
