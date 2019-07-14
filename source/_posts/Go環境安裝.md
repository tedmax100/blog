---
title: Go環境安裝
date: 2019-07-14 15:30:26
categories: "Go"
tags:
    - Go
---
![](https://golang.org/lib/godoc/images/go-logo-blue.svg)  
[Download page](https://golang.org/dl/)
[Install doc](https://golang.org/doc/install)

### Install the GO on Linux
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

#### First Go Program
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
	fmt.Println("hello")
}
```

```bash
# 編譯產生可執行的二進制檔案, 會被安裝到$GOPATH/bin底下
go install hello
# 執行
$GOPATH/bin/hello
> hello
```