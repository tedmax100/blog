---
title: Go Testing初探
date: 2020-12-21 21:39:23
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
程式寫好了!!
來稍微測試自己的程式會不會跑.
但Go只有main包的main()才能執行阿!!

還是要寫另一個專案的程式來測試剛剛寫的程式呢?

Go內建測試框架[testing](https://golang.org/pkg/testing/)
讓你可以把想寫的測試程式寫在裡面, 透過`go test`來執行測試.
​<!-- more -->
# 單元測試[Unit Test](https://zh.wikipedia.org/wiki/%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95)
通常會利用testing來寫所謂的`單元測試UnitTest`.
用來測試某一個package, 或是某一段程式碼甚至是某一個函數.

單元測試的單元指的是人為規定的最小的被測功能模組.
(恩, 打完這句話我自己也看不懂.)
廣義的說就是該被測功能的內部組成, 該組成能是`stucts`、有接收器的`method()`跟一些global的`func()`, 這些都能被叫做是**單元**.
這些單元要夠可靠、有效率, 這樣組合起來的模組才會一樣可靠有效率.

所以透過unit test來協助我們來度量這些維度.

先來寫一個FizzBuzz的功能.
一個班級有一堆學生(廢話)
老師指向隨機一個同學,  要他從1開始講, 然後下一位同學接著講下一個數字.
只要自己的數字能夠被3給整除, 就講Fizz ; 能被5整除講Buzz
要是該數字能被3跟5給整除, 就講FizzBuzz

`fizzBuzz.go`
```go=1
package main

import "strconv"

func FizzBuzz(num int) string {
	if num%15 == 0 {
		return "FizzBuzz"
	} else if num%3 == 0 {
		return "Fizz"
	} else if num%5 == 0 {
		return "Buzz"
	}
	return strconv.Itoa(num)
}
```
`main.go`
```go=1
package main

import (
	"fmt"
)

func main() {
	var studentAmount int = 50
	for number := 1; number <= studentAmount; number++ {
		fmt.Println(FizzBuzz(number))
	}
}
/*
Buzz
11
Fizz
13
14
FizzBuzz
16
*/
```
m...怎確認對不對呢? 用肉眼檢查？
要是你想到是這樣的方式!!  那你還`不夠懶`!! 太勤勞了XD


首先先建立一個fizzBuzz_test.go的檔案.
測試方法的參數都要是*testing.T.
方法名稱也要是Test開頭.

testing.T提供的操作方法:
### Log、Logf
列印出Log, 並且結束測試
Logf是加上格式化的功能
```go
func (c *T) Log(args ...interface{})
func (c *T) Logf(format string, args ...interface{})
```
### Error、Errorf
列印出錯誤Log, 並且結束該案例的測試, 往下一個案例前進; 測試結果會是Fail
```go
func (c *T) Error(args ...interface{})
func (c *T) Errorf(format string, args ...interface{})
```

### Fatal、Fatalf
列印出錯誤Log, 並且中斷測試; 測試結果會是Fail
```go
func (c *T) Failed() bool
func (c *T) Fatal(args ...interface{})
```

## Table-Driven Test
使用表驅動測試, 就是給一組列表內有輸入和預期的結果;
都去執行相同的待測單元, 然後比對預期的結果.
未來有新的case加進去就好.

這裡我懶,  就跑一組
```go=1
package main

import (
	"testing"
)

func TestFizzBuzzSuccess(t *testing.T) {
	var testCases = []struct {
		in  []int
		out []string
	}{
		{
			in:  []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15},
			out: []string{"1", "2", "Fizz", "4", "Buzz", "Fizz", "7", "8", "Fizz", "Buzz", "11", "Fizz", "13", "14", "FizzBuzz"},
		},
	}

	for idx := range testCases {
		for i := range testCases[idx].in {
			if testCases[idx].out[i] != FizzBuzz(testCases[idx].in[i]) {
				t.Errorf("not correct, input: %s ; output: %s", FizzBuzz(testCases[idx].in[i]), testCases[idx].out[i])
			}
		}
	}
}
```
```bash
go test
-----
PASS
ok      UnitTest     0.001s
```

錯誤也是一種測試情境, 當我們給錯誤資料或錯誤行為時, 本來就是預期返回錯誤.


## go test
go test默認行為是執行所有的測試範例.
加上`-run`可以指定單個測試範例執行.
```bash
go test -run TestFizzBuzzSuccess
```
單元測試應該都要極快完成, 這樣我們未來改動一點點程式時, 都能跑一下單元測試.
不然改個幾個字, 要等1分鐘...生命就流逝去了.
但要怎麼夠快, 簡單的說就是只有測試邏輯, 不測試外部依賴跟資料.
外部依賴跟資料全作假的, 自然就會快了.
時間也是種外部依賴, 不然排程服務每次測試都要等時間到, 也是慢.

### 加上超時限制
-timeout 時間
```bash
go test -run TestFizzBuzzSuccess -timeout 1ns 
---------
panic: test timed out after 1ns
```

[VsCode插件 - Go Test Explorer](https://marketplace.visualstudio.com/items?itemName=premparihar.gotestexplorer)

# Benchmark Test
Benchmark這個在遊戲測試報告中應該常聽到的名詞.
這裡是用來取得待測目標`執行效率`和`記憶體佔用`的情況作測試.

這裡測試方法名稱要以Benchmark開頭, 傳入參數為(b *testing.B)
b.N表示的是循環的次數, 因為是壓測, 所有要反覆的測試待測目標

測試時間預設是1秒鐘, 會顯示測試次數跟每次所花費的時間成本.

```go=1
func Benchmark_FizzBuzz(b *testing.B) {
	for i := 0; i < b.N; i++ {
		FizzBuzz(i)
	}
}
```
### 效率測試
```bash
/*
go test -bench=.    
----------------------
goos: linux
goarch: amd64
pkg: UnitTest
Benchmark_FizzBuzz-4    50000000                22.8 ns/op
PASS
ok      UnitTest     1.166s
*/
```
總共測試50000000次, 每次大約花費22.8 ns.

還能自己定義測試時間
加上-benchtime
```bash
go test -bench=. -benchtime=2s
-------
goos: linux
goarch: amd64
pkg: UnitTest
Benchmark_FizzBuzz-4    100000000               22.6 ns/op
PASS
ok      UnitTest     2.292s
```

### 記憶體佔用測試
-benchmem
```bash
go test -bench=.  -benchmem
-------
goos: linux
goarch: amd64
pkg: UnitTest
Benchmark_FizzBuzz-4    50000000                22.7 ns/op             4 B/op          0 allocs/op
PASS
ok      UnitTest     1.165s
```
4B/op 表示每次呼叫呼叫配置4Bytes
0 allocs/op 表示每次呼叫需要進行幾次分配對象記憶體.
這裡是0 是因為我們的待測目標沒有去new或是宣告任何參考型別.

### 輸出profile報告
-cpuprofile 檔名.out // 每10ms採集一次CPU使用情況
-memprofile 檔名.out // 執行期間, heap的分配情況
-memprofilerate n // n表示取樣間隔, 每當有n個Bytes的記憶體被配置時, 就會採樣紀錄一次
-blockprofile 檔名.out //記錄Goroutine發生的阻塞事件
-blockprofilerate n //記錄goroutine發生阻塞事件的時間間隔, n為次數, 預設1次.


# 匯出測試的覆蓋率
-cover 啟動覆蓋率分析
-coverprofile 檔名.out // 把所有通過測試的覆蓋率報告寫檔
![](https://i.imgur.com/Z9DjTgq.png)
把剛剛的coverprofile給圖像化
透過go tool來顯示, 方便我們理解哪裡沒有被測試到.
```bash
go tool cover -html=cov.out
```
![](https://i.imgur.com/OtnruNd.png)
![](https://i.imgur.com/840JgTc.png)


> go還有強大的pprof的功能, 可以把各種上面的報表給圖像化的功能.
> 方便我們找出效能問題在哪裡.
![](https://i.imgur.com/cP4A9F4.png)


[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10221362)