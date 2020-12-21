---
title: 好多依賴要管理
date: 2020-12-21 21:17:18
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](https://cdn-images-1.medium.com/max/800/1*nPaOoEou7T-cM9WZGtU-gg.png)

回憶一下之前[Day01](https://ithelp.ithome.com.tw/articles/10214347)提到的
​<!-- more -->
# Go WorkSpace 工作目錄
我們安裝好Go之後進去預設的GOPATH目錄下, 就會看到這樣的目錄結構.
```
- GOPATH
    |
    -- bin/
    |
    -- pkg/
    |
    -- src/
        |
        -- project1/
                |
                -- vendor/
        |
        -- project2/
                |
                -- vendor/
```
- bin 包含可安裝並執行的command (可執行的二進制文件)
- pkg 包含各種package objects (二進制的library檔, *.a檔)
- src 包含各專案的代碼

## GOPATH
GOPATH是一個環境變數, 用絕對路徑來指定工作目錄的位置.
要是我們多人參與開發, 每個人都有一套自己的目錄結構, 讀取配置文件的位置也不統一, 這樣輸出的二進制文件也不會統一, 會導致開發的標準不一.

GOPATH存在的目的是   
- 所有在Go代碼裡, 透過import 宣告的package path, 用來計算該包的路徑用.
- 儲存任何透過go get獲取的依賴包.
- go build、go install產生的二進制文件會放在$GOPATH/bin底下

## go get
官方提供的工具, 會把go get取得的第三方套件代碼存放到$GOPATH/src中.

有許多社群做了幾個package management工具 Glide、dep、 govendor, 包含後面出的gomodule等, 都是為了方便專案去管理使用了哪些依賴包跟對應的版本, 以及下載位置.
小弟接觸比較晚, 就挑了govendor和gomodule來學習.
這兩個可以共存XD

# vendor
在Go Module還沒出來時, 在1.5版提供了vendor. 但要手動環境變數`GO15VENDOREXPERIMENT=1`
1.6版則是默認是1
1.7版則是不必再設定該環境變數, 默認開啟vendor

## vendor特性
在我們執行go build 或者是go run時, go會依照下列順序依序去找我們的要的依賴包
1. 當下專案目錄的vendor資料夾
2. 一路往上層目錄查找, 直到找到$GOPATH/src下的vendor
3. 在GOROOT目錄下查找
4. 在GOPATH下查找

## vendor使用建議
- 一個專案只會有一個vendor目錄, 且就位於專案的根目錄內.

# govendor
govendor就是一個基於vendor這種目錄機制所做出來的套件管理工具.
go在以前常用的套件包管理工具其中之一就是govendor.
能在go build時的應用路徑搜尋調整成為`當前專案項目目錄/vendor`目錄的方式.


## 安裝govendor
```bash
go get -u -v github.com/kardianos/govendor
```
![](https://i.imgur.com/Sr2pAem.png)
安裝好到$GOPATH/bin下, 會看到govendor的可執行檔.

## 使用govendor
### 初始化vendor
```bash
// 移動該專案的根目錄
govendor init
```
![](https://i.imgur.com/OctoOpo.png)

### 下載依賴包
#### 下載master主幹下最新的commit
```bash
govendor fetch 路徑
```
![](https://i.imgur.com/od7jm47.png)
vendor.json 
用來紀錄依賴包的commit的hash跟時間等等
![](https://i.imgur.com/YfB5JNs.png)
![](https://i.imgur.com/RD7nIqv.png)

#### 下載特定的版本
```bash
govendor fetch 路徑@v版本
```
![](https://i.imgur.com/XbMwHdF.png)
![](https://i.imgur.com/MfEkr9v.png)
vendor.json 
用來紀錄依賴包的列表版本, commit的hash跟時間等等
![](https://i.imgur.com/z0GErSe.png)

#### 下載特定的tag 或是branch
```bash
govendor fetch 路徑@=tag_name
govendor fetch 路徑@=branch_name
```
![](https://i.imgur.com/exzxBvY.png)
![](https://i.imgur.com/fWwl0ON.png)
![](https://i.imgur.com/heJ1u5l.png)


#### 加入GOPATH現有的包到vendor管理下
##### 從GOPATH下複製指定的包
```bash
govendor add path
```
這依賴包位於我的$GOPATH/src/githut.com/xwb1989/sqlparser目錄下
![](https://i.imgur.com/P2ud12L.png)

##### 添加所有的依賴包
```bash
govendor add +external
```
![](https://i.imgur.com/wQLVbcD.png)

##### 使用自己小改過的包來取代官方第三方依賴包
可能內部對github.com/go-sql-driver/mysql有加入點東西, 就能用這種方式改用自己的,但是程式import 還是照常github.com/go-sql-driver/mysql
```bash
govendor get 'github.com/go-sql-driver/mysql::github.com/tedmax100/go-mysql'
```

### 刪除沒用的依賴包
```bash
govendor remove +unused
```
輸入完之後, 會發現全空, 因為我這時該專案目錄下還沒有任何程式作import.
![](https://i.imgur.com/jyz0TeM.png)

```go
package main

import (
	log "github.com/sirupsen/logrus"
)

func main() {
	log.WithFields(log.Fields{
		"animal": "walrus",
	}).Info("A walrus appears")
}
```
透過govendor再次安裝依賴包
```bash
govendor fetch github.com/sirupsen/logrus@v1.4.2  
govendor add github.com/xwb1989/sqlparser/ 
```
![](https://i.imgur.com/UGLfP5z.png)

```bash
govendor remove +unused
```
![](https://i.imgur.com/JPtgky3.png)
只會留下有用到的. 

再次清空govendor所有依賴
執行add
```bash
govendor add +external
```
也會得到跟上圖一樣的結果.

```bash
go run main.go
```
![](https://i.imgur.com/r4VcKSZ.png)
成功執行!

### 列出該專案所有存在的依賴包
```bash
govendor list
```
![](https://i.imgur.com/sj78URu.png)

### 從vendor.json恢復所有依賴包原始碼到vendor目錄下
```bash
govendor sync
```

# 與npm、yarn相同的使用命令
|command|npm|yarn|govendor|
|:-----:|:-:|:--:|:------:|
|初始化|npm init|yarn init|govendor init|
|增加依賴包|npm install -s|yarn add|govendor fetch|
|刪除依賴包|npm uninstall|yarn remove| govendor remove|
|同步依賴包|npm install|yarn install|govendor sync|


# govendor看似完美了, 但幹麻還出gomodule?
因為govendor要求一定要在$GOPATH/src下執行.
不然會報錯誤
![](https://i.imgur.com/kmdeQ7p.png)

下一篇的go module就是能解決這問題

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10216807)