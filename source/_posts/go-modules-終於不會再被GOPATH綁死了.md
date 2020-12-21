---
title: Go modules 終於不會再被GOPATH綁死了
date: 2020-12-21 21:18:29
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
# Go Modules

## Go modules 出現原因
1. 解除對GOPATH的完全依賴, 有go modules就能在$GOPATH外開專案了.
2. 不同環境或者是多專案, 需要一套切換vendor目錄.
3. 同一個依賴包的多種版本共存問題, 加入了版本化的支援.
4. 可以使用GOProxy來解決某些地區無法使用go get的問題.
5. 以往需要將vendor目錄一起提交到git, 避免CI/CD去拉到外部的依賴包.
6. go modules有build cache, 在CI build server上速度飛快.
​<!-- more -->

## 環境準備
* Go version >= 1.11
* GO111MODULE=on (Go MOdule模式), 使用go module, 不諮詢GOPATH, 只是下載下來的依賴包依然存在GOPATH/pkg/mod/底下.
> GO111MODULE=off, 這表示是GOPATH模式, 查找依賴包順序如同昨天提的vendor目錄和GOPATH下.  

> GO111MODULE=auto, 默認模式,在這模式下要使用go module, 需要滿足兩個條件
> 1. 該專案目錄不在GOPATH/src/下
> 2. 當前或上一層目錄存在go.mod檔案
## Go Mododules 對於匯入依賴包的影響 
* 可以在$GOPATH之外的地方建立專案
* 該專案Go Module開啟後, 下載的package會放在$GOPATH/pkg/mod下.
![](https://i.imgur.com/Op2TRi6.png)
* $GOPATH/bin的功能依然保持

### Go Mod Commands
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

### go.mod的一些名詞
* module 
    * 定義模組路徑
* go
    * 定義預期的go version
* require
    * 指定依賴的功能包和其版本或是更高版本[預設是最新版]
* exclude
    * 排除該功能包和其版本
* replace
    * 使用不同的依賴包版本替換原有的依賴包版本
* 註解 
    * // 單行註解
    * /* 多行註解 */
    * indirect 被間接導入的依賴包
```
module my/package
go 1.12
require other/thing v1.0.2 // 註解
require new/thing/v2 v2.3.4 // indirect
exclude old/thing v1.2.3
replace bad/thing v1.4.5 => good/thing v1.4.5
```

### 同專案的子目錄
因為go.mod在專案的根目錄下, 子目錄的導入路徑會是該專案的導入路徑+子目錄路徑.
舉例: 建立了ithome的專案, 底下有一個ironman的子目錄.
則不需要也在子目錄建立go mod init指令, Go build會自動辨識ironman這目錄是ithome的一部分.

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

**gomaintest**
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


## 自己寫個共用依賴模組用在自己的專案試試看
### 依賴包專案
目錄結構 /GOPATH/src/ithome
![](https://i.imgur.com/kwKItNC.png)
```bash
go mod init github.com/tedmax100/ithome
```
因為我等等要推上github的repo中, 這裡就如以前說的會有域名/目錄/專案...
這樣的層次關係.
```bash
go get github.com/sirupsen/logrus
```
![](https://i.imgur.com/FiWIWFs.png)
這裡跟govendor fetch有些不同了, 再有go modules專案內輸入go get. 
預設會去抓最新的tag版本; 如果沒有設立tag, 就抓最新的commit版本.

go.sum這時候就會把logrus目錄下go.mod跟go.sum的依賴包跟其版本保存起來.
go.sum 其實跟npm的package-lock.json有著一樣的功能.

go.mod(npm的package.json)定義我們指名要的依賴跟版本.
go.sum把go.mod的所有依賴包,  每一個像是樹的根節點一樣, 開始走訪去下載,  並且紀錄關係在此.

**ironman/ironman.go**
```go=1
package ironman

import (
	log "github.com/sirupsen/logrus"
)

func PrintIronMan() {
	log.Info("hi iron man")
}
```
**ithome.go**
```go=1
package ithome

import (
    // 這裡因為我們定義的mod name就這麼長, 
    // 子目錄的導入路徑會是該專案的導入路徑+子目錄路徑.
	"github.com/tedmax100/ithome/ironman"

	log "github.com/sirupsen/logrus"
)

func PrintItHome() {
	log.Info("hi ItHome")
	ironman.PrintIronMan()
}
```
存檔, commit, 推上github.
這裡我沒有打release tag.


### 可執行的專案
目錄結構 /GOPATH/src/gomod
![](https://i.imgur.com/9ktoRgl.png)

```bash
go mod init gomod
// 下載依賴包
go get -u github.com/tedmax100/ithome
```
![](https://i.imgur.com/pWoTXF4.png)

![](https://i.imgur.com/t3Ivc77.png)

**main.go**
```go=1
package main

import (
	"github.com/tedmax100/ithome"
)

func main() {
	ithome.PrintItHome()
}
```
執行main.go
![](https://i.imgur.com/BY0VXSv.png)

## 把依賴包給作個release tag, 試試看
![](https://i.imgur.com/dkkYARP.png)

```bash
// 作個更新
go get -u github.com/tedmax100/ithome
```
![](https://i.imgur.com/xApGu57.png)
![](https://i.imgur.com/phh39lT.png)
可以看到ithome這依賴包, 從本來是紀錄commit hash, 變成是紀錄tag版本號了.

## 把依賴包給再進個commit, 但tag 還在v0.0.1
![](https://i.imgur.com/li9NXnE.png)

```bash
// 作個更新
go get -u github.com/tedmax100/ithome
```
![](https://i.imgur.com/yZak3mi.png)
正如前面說的, 他會先找tag/release有沒有, 沒有才去找最新的commit.
但因為我們已經有tag v0.0.1, 所以怎樣更新依賴, 
只要沒有更新版的依賴被release就不會被更新.

## 那! 就來進版吧
![](https://i.imgur.com/vuB48xY.png)
![](https://i.imgur.com/5y7NWRb.png)

各版本有下載過得都會在go/pkg/mod/匯入包路徑底下
![](https://i.imgur.com/gOntitp.png)

## 反悔了! 想退回去指定的某一版
```bash
go get github.com/tedmax100/ithome@v0.0.1
```
因為快取有了, 就不必重抓
![](https://i.imgur.com/ceLjbx5.png)
也會順便更改go.mod和go.sum的內容
![](https://i.imgur.com/3aKN9Y8.png)

## 這外部的難用, 我要用自己魔改過得, 放在vendor底下的
> 或 我怕外部有人偷偷在代碼放後門, 我要用自己網路cache有的, 複製到vendor下
```bash
go mod vendor
```
這會建立出一個vendor目錄, 底下有現在go.mod依賴包的代碼.

我們改一下程式
**gomod/vendor/github.com/tedmax100/ithome/ithome.go**
```go
package ithome

import (
	"github.com/tedmax100/ithome/ironman"

	log "github.com/sirupsen/logrus"
)

func PrintItHome() {
    // 就改這行, 存檔
	log.Info("hi ItHome from vendor")
	ironman.PrintIronMan()
}

```

開心的在terminal輸入
```bash
go run main.go
```
![](https://i.imgur.com/VEhql7D.png)

笑XD
因為只要啟用了go modules, 就會完全忽略了vendor目錄的存在, 只讀取go.mod的內容.

那怎辦呢?
原本的指令go build, go install, go runm, go test啦 
等等的加上`-mod=vendor`

![](https://i.imgur.com/Mrm4fMm.png)

## 多安裝一些依賴包
```bash
go get github.com/go-sql-driver/mysql   
```
![](https://i.imgur.com/GnChMVz.png)

結果最後根本沒有半個地方有import 
怎辦, 自己檢查每一個.go檔案, 看哪些沒有import ?

哪些依賴又沒有抓到呢?

```bash
# add missing and remove unused module
go mod tidy
```
![](https://i.imgur.com/9K6mvKj.png)

## 依賴包的module名稱能不能帶上版本號?
> 要是有breaking change, 新舊版本無法兼容呢?
**ithome/go.mod**
```
module github.com/tedmax100/ithome@v2.0.0  // 這裡打上版本號

go 1.12

require github.com/sirupsen/logrus v1.4.2
```
改個程式
```go=1
package ithome

import (
	"github.com/tedmax100/ithome/ironman"

	log "github.com/sirupsen/logrus"
)

func PrintItHome() {
	log.Info("hi ItHome V0.0.7")
	ironman.PrintIronMan()
}

func PrintItHomeV2() {
	log.Info("hi ItHome V2.0")
	ironman.PrintIronMan()
}
```
存檔commit, push作release

跑到執行專案, 執行
```bash
go get -u github.com/tedmax100/ithome
```
這時候發現, 不會去下載這2.0.0版本的依賴包
![](https://static.studygolang.com/190620/8e56f0a679b7f993b3354aa70f779da1.png)
因為版本號的v2.0.0, 這個第一個數字表示主版本號, 不同版本間若是無法兼容使用, 
則建議是提昇這版本號, 且建議遠端分之多上v2分支.
版本號若是v1.10.13, 這個1表示主要版本號, 10表示次要版本號,  13表示修正版本號
且go get -u會檢查go mod的版本號, 並不會主動去下載並提昇到不同的主要版本號的依賴包.

這裡import改成使用v2版
```go
package main

import (
	"github.com/tedmax100/ithome/v2"
)

func main() {
	ithome.PrintItHome()
	ithome.PrintItHomeV2()
}
```
```bash
go mod tidy
```
![](https://i.imgur.com/OdaAcQJ.png)


**開心了, 收工**

go mod 可以相當完美的跟vendor做切換並存.
有機會來玩玩看goproxy.


[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10217414)