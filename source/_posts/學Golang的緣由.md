---
title: 學Golang的緣由
date: 2020-12-20 16:17:50
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](https://i.imgur.com/DNWjAse.gif)
### 學Golang的緣由
<!-- more -->
這是小弟第一次參加鐵人賽, 來挑戰一下自我.
開始學著寫Golang的原因是因為寫了幾年NodeJS跟C#, 
但Node真的一個專案打包成docker image超臃腫.
就嘗試找一個也支援高併發, 性能優, 方便部屬的語言, 
但希望它的執行檔大小能是超小的, 且各種OS都支援.
就選擇Golang這語言了.
就下班加減學一點學一點, 至今也看了兩三個月.
一些東西紀錄在[自己的部落格](https://tedmax100.github.io/)當作筆記


### Go語言特性
- Google開發並負責維護的開源專案!
-  靜態、編譯型, 自帶GC和併發處理的語言, 能編譯出目標平台的可執行檔案, 編譯速度也快.
-  全平台適用, [Arm](https://github.com/golang/go/wiki/GoArm?fbclid=IwAR0Hz1xEpTuLCVoxJhSGY_rRj0ivwITuQr6cezd8elYMeLJu7-P4mSIiY5E)都能執行
-  上手容易, 我覺得跟C比較真的頗容易,  但跟JS比我覺得還是差一些
-  原生支援併發 (goroutine), 透過channel進行通信
-  關鍵字少, 30個左右吧
-  用字首大小寫, 判別是否是public / private
-  沒用到的import 或者是 變數, 都會在編譯時期給予警告
-  沒有繼承!
- 適合寫些工具, 像是[hugo](https://gohugo.io/)、[fzf](https://github.com/junegunn/fzf)、[Drone](https://github.com/drone/drone)、[Docker](https://github.com/docker)
- 適合其他語言大部分的業務, RestAPI, RPC, WebSocket
- 內含[測試框架](https://golang.org/pkg/testing/)
- 不必在煩惱 到底要i++還是++i了, 因為在Go裡沒有++i, 也不能透過i++賦值給其他的變數

### 從Node到Golang
#### Hello World 
NodeJS
```javascript=1
console.log("hello world");
```
```bash
> node app.js
```

Golang的對等寫法
```go=1
package main
import (
    "fmt"
)

func main() {
    fmt.Println("hello world")
}
```
```bash
> go run main.go
```

#### Array 和 Slice
```javascript=1
const names = ["it", "home"];
```

```go=1
names := []string { "it", "home"}
```

##### 印出後面幾個字的子字串
```javascript=1
let game = "it home iron man";
console.log(game.substr(8, game.length));
```
```go=1
game := "it home iron man"
fmt.Println(game[8: ])
```

### 流程控制
```javascript=1
const gender = 'female';

switch (gender) {
    case 'female':
        console.log("you are a girl");
        break;
    case 'male':
        console.log("your are a boy");
        break;
    default:
        console.log("wtf");
}
```
```go=1
gender := "female"
switch gender {
case "female":
    fmt.Println("you are a girl")
case "male":
    fmt.Println("your are a boy")
default:
    fmt.Println("wtf")
}
```
看得出來Go省略了break這關鍵字

### Loop
Javascript有for loop, while loop, do while loop
Go只有for loop 就能模擬上面三個
```go=1
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// key value pairs
kvs := map[string]string{
    "name":    "it home",
    "website": "https://ithelp.ithome.com.tw",
}

for key, value := range kvs {
    fmt.Println(key, value)
}
```

### Object
```javascript=1
const Post = {
    ID: 10213107
    Title: "下班加減學點Golang",
    Author: "Nathan",
    Difficulty: "Beginner",
}
```
```go=1
type Post struct {
  ID int
  Title string
  Author string
  Difficulty string
}

p := Post {
  ID: 10213107,
  Title : "下班加減學點Golang",
  Author: "Nathan",
  Difficulty:"Beginner",
}
```
Go能透過定義抽象的struct與其屬性, 在實例化
也能透過map[string]interface來定義
```go
Post := map[string]interface{} {
  "ID": 10213107,
  "Title" : "下班加減學點Golang",
  "Author": "Nathan",
  "Difficulty":"Beginner",
}
```

從上面幾個例子就能看的出來Node跟Go語法結構上很類似, 
所以學過Node再來學Go好像就沒那麼難了 XD
之後會慢慢補充Go的更多東西. 

謝謝各位

[下班加減學點Golang與Docker-鐵人賽連結](https://ithelp.ithome.com.tw/users/20104930/ironman/2647)

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10214255)