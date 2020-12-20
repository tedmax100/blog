---
title: 第一隻Go程式
date: 2020-12-20 16:50:06
tags:
    - Go
    - iT邦鐵人賽11Th
---
# 安裝Go跟開發環境
[Golang下載](https://golang.org/dl/)
[Install doc](https://golang.org/doc/install)
[VsCode](https://code.visualstudio.com/)

## Install the GO on Linux
```bash=
# Download file
wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
# Extract it into /usr/local
tar -C /usr/local -xzf go1.12.7.linux-amd64.tar.gz
# Add /usr/local/go/bin to the Path environment variable
export PATH=$PATH:/usr/local/go/bin

# Check installation
go env
```
![goenv](https://i.imgur.com/j62hYTR.png)
其他名稱會在後面講package時會稍微提到.

### Upgrade Go
```bash
# Download file
wget https://dl.google.com/go/go$VERSION.linux-amd64.tar.gz
# Extract it into /usr/local
tar -C /usr/local -xzf go$VERSION.linux-amd64.tar.gz
# Add /usr/local/go/bin to the Path environment variable
export PATH=$PATH:/usr/local/go/bin
```

#### Upgrade by shell script
[update-golang](https://github.com/udhos/update-golang)


### Workspaces
[Workspaces](https://golang.org/doc/code.html#Workspaces)
[Setting GoPath](https://github.com/golang/go/wiki/SettingGOPATH)
在GoPath所顯示的目錄下創建以下資料夾
* src : go source file
* pkg : 編譯產生的文件, .a檔案(一包object file) ; 暫態緩存文件
* bin : 編譯後可執行檔案
```bash
mkdir -p $GOPATH/src $GOPATH/pkg $GOPATH/bin
```

### Hello Go
```bash
mkdir -p $GOPATH/src/hello
cd $GOPATH/src/hello
code .
```
以VsCode開啟該目錄

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello Go")
}
```
```bash
# 編譯產生可執行的二進制檔案, 會被安裝到$GOPATH/bin底下
go install hello
# 執行
$GOPATH/bin/hello
> Hello  Go
```

##  Main package
Go每支檔案都會需要宣告這是屬於哪個package的, 相當於C#的namespace概念.
主要的會有一個叫做main的package包, 做為這隻可執行程式的入口包. 
如果該專案沒有main包時, 就沒法被編譯成可執行檔案. 
所以如果是要做成共享套件, 就可以不必有main包的存在於該專案內. 

main裡面會有main方法作為程式的執行進入點.
```go=1
// main包宣告
package main

// 匯入fmt包
import (
  "fmt"
 )
 
 // main 方法, 作為執行程式的入口
 func main() {
    fmt.Println("Hello IThome")
 }
```

**import**  
用來導入其他的包, 要用雙引號作為字串來使用.
- 單行匯入
```go
import "包A"
import "包B"
```
- 多行匯入, 宣告順序不影響真正的匯入結果
```go
import (
  "包A"
  "包B"
)
```

要是我有一個包在$GOPATH/src/底下的資料夾路徑是這樣的
* github.com
    * ithome
        * packageA
那我要引入 packageA的話要按照$GOPATH開始計算的路徑, 使用/進行路徑分隔.
也因為跟資料夾路徑有關, 所以建議上都是把資料夾名稱跟package名稱取名成一致.
```go
import (
  "github.com/ithome/packageA"
)
```

### 安裝第三方套件
今天想安裝mysql套件, 他的遠端路徑是 github.com/go-sql-driver/mysql
依照 /作路徑分隔的話.
第一段表示網域名稱
第二段表示作者或者是機構名稱
第三段則是專案名稱

透過go get指令, 透過這指令下載原始碼並且編譯.
由於go get需要GOPATH已經被設置, Go1.8之後GOPATH預設在用戶目錄的go資料夾下.

```bash
go get  github.com/go-sql-driver/mysql
```

go get 參數說明:
- -d 只有下載, 不會安裝
- -v verbose, 顯示下載編譯時的log
- -u 更新既有的依賴包


有了基本包的概念, 就能寫簡單的範例了.
```bash
# 安裝logrus這log套件
go get github.com/sirupsen/logrus
```
**go/src/packagedemo/mylib/add.go**
```go=1
package mylib

func Add(a, b int) int {
  return a + b
}
```

**go/src/packageDemo/main.go**
```go=1
package main
import (
  "fmt"
  "packagedemo/mylib"
  // 這裡使用log 這別名來取代logrus這包名
  log "github.com/sirupsen/logrus"
)

func main() {
  fmt.Println(mylib.Add(1,2))
  log.Info("IThome Iron man")
}
```
執行
```bash
go run main.go
# 輸出 :
# 3
# INFO[0000] IThome Iron man  
```