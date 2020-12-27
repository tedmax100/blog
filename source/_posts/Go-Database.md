---
title: Go database/sql, 和資料庫打個招呼
date: 2020-12-21 21:37:15
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
# SQL
在做專案時, 都會需要關聯式資料庫做資料的CRUD.
Go提供了database/sql包來讓開發者跟資料庫打交道, 這包就像Java的JDBC.
database/sql包只是定義了一套操作資料庫的接口和抽象層定義.
所以還是需要實體的驅動, 這裡我選用[MySQL](https://github.com/go-sql-driver/mysql).
[各種Go SQL Drivers](https://github.com/golang/go/wiki/SQLDrivers)
我們開發者幾乎都是在操作database/sql包所提供的接口方法而已.
大部分情境, 都只要在程式的某地方設定好驅動就好.
​<!-- more -->
#### 下載MySQL驅動包
```bash
go get github.com/go-sql-driver/mysql
```

# [database/sql](https://golang.org/pkg/database/sql/)
來認識幾個type, 因為我在這撞牆蠻久的.
C#跟Node套件用習慣了QQ

## [DB](https://golang.org/pkg/database/sql/#DB)
資料庫的物件實例, 同時內部也有自己實現的連線池, 其池是一個包含多個open和idle連線的池子.
使用時被選到的連線會被標記成`open`, 完成後會被標記成`idle`
他需要有driver打開或關閉資料庫, 管理連線池.
同時它也是線程安全的, 可以不必重複創立, 只需要一個就可以傳遞給多個goroutine使用.
![](https://i.imgur.com/y47gGlS.jpg)
一個泳池的泳道數量設定好之後就是固定的, 只要有人使用, 該泳道就被open, 等到有人離開泳道, 該泳道就被視為idel.
maxlifetime, 就視為該泳道的開放使用時間吧, 也許設置成1小時換水清理一次.
但有人還在用的話, 當然就要等它被釋放出來, 才能開始清潔. (有點牽強的例子)

![](https://i.imgur.com/fj3fOcS.png)

 * ### SetMaxIdleConns
設定空閒連線池的最大連線數
```go
func (db *DB) SetMaxIdleConns(n int) {...}
```

 * ### SetMaxOpenConns
設定最大打開的連線數, 0表示不限制.
```go
func（db * DB）SetMaxOpenConns（n int） {...}
```

 * ### SetConnMaxLifetime
設定連線可重複利用的最大時間長度, 0是預設值表示沒有max life, 總是可重複使用.
```go
func (db *DB) SetConnMaxLifetime(d time.Duration) {...}
```
```go
db.SetConnMaxLifetime(time.Hour)
// 設定每一個連線最大生命週期1hr
// 
```

 * ### Open
```go
func Open(driverName, dataSourceName string) (*DB, error) {...}
```
初始化一個sql.DB對象, 但還沒真正建立連線.
會啟動一個connectionOpener的goroutine. 
也初始化一個openerCh channel.
需要連線時, 就對該channel發送數據就好.
```go
db := &DB{
		driver:   driveri,
		dsn:      dataSourceName,
		openerCh: make(chan struct{}, connectionRequestQueueSize),
		lastPut:  make(map[*driverConn]string),
	}
```
在這兩種情形下會去建立連線 :
1. 會在第一次呼叫ping(), 真正建立連線
2. 呼叫db.Exec()或者db.Query(), 如果空閒連線池有連線, 就直接取用; 如果沒有就會產生一個新的連線.



## [Config](https://godoc.org/github.com/go-sql-driver/mysql#Config)
go-sql-driver/mysql所提供的結構體, 有提供幾個針對DSN的方法
```go
// config的建構式
func NewConfig() *Config
// 把dsn字串剖析轉成config物件
func ParseDSN(dsn string) (cfg *Config, err error)
// 複製一個config物件的副本
func (cfg *Config) Clone() *Config
// 把config結構體格式化成dsn格式的連線字串
func (cfg *Config) FormatDSN() string
```
## [Row](https://golang.org/pkg/database/sql/#Row)
呼叫QueryRow()之後返回的單行結果
掃描取得的結果到dest上; 但如果有多個結果, scan做完第一行後, 就會丟棄其他行的資料了.
如果row是沒有資料的, 則是會返回ErrNoRows這錯誤.
```go
func (r *Row) Scan(dest ...interface{}) error {...}
```
## [Rows](https://golang.org/pkg/database/sql/#Rows)
查詢的結果, 會有個指標從開始直到結束.
呼叫Next(), 就去取得下一行row的資料.
```go
// 主要是用來讓連線釋放回連線池
func (rs *Rows) Close() error {...}
// 對資料做走訪,正常結束的話內部會自動呼叫rows.Close()
func (rs *Rows) Next() bool
// 切換到下一個結果集, 一次查詢是可以返回多個結果集的
func (rs *Rows) NextResultSet() bool
// 
func (rs *Rows) Scan(dest ...interface{}) error
```
## [Scanner](https://golang.org/pkg/database/sql/#Scanner)
```go
type Scanner interface {
    // Scan assigns a value from a database driver.
    Scan(src interface{}) error
}
```
## [Stmt](https://golang.org/pkg/database/sql/#Stmt)
查詢語句, DDL、DML等的prepared sql語句.
## [Tx](https://golang.org/pkg/database/sql/#Tx)
一個進行中的資料庫事務.
呼叫db.Begin()之後, 會取得一個tx物件, 需要呼叫Commit()或Rollback()才會結束事務.並且歸還連線回連線池.

## [Result](https://golang.org/pkg/database/sql/#Result)
主要是針對insert、update、delete的操作所返回的結果.
Result是個接口,  定義了LastInsertId()和RowsAffected().
各驅動會實作這接口的兩個方法.
```go
type Result interface {
    LastInsertId() (int64, error)
    RowsAffected() (int64, error)
}
```
MySqlDriver的實作:
```go
package mysql

type mysqlResult struct {
	affectedRows int64
	insertId     int64
}

func (res *mysqlResult) LastInsertId() (int64, error) {
	return res.insertId, nil
}

func (res *mysqlResult) RowsAffected() (int64, error) {
	return res.affectedRows, nil
}
```
## [Nullable Property](https://golang.org/pkg/database/sql/#NullBool)
各種允許null的基礎型別. 且都有實現Scanner的Scan().
## [IsolationLevel](https://golang.org/pkg/database/sql/#IsolationLevel)
事務用的[資料隔離等級](https://myapollo.com.tw/2018/09/30/database-transaction-isolation-levels/).
```go
const（
    LevelDefault  IsolationLevel = iota 
    LevelReadUncommitted 
    LevelReadCommitted 
    LevelWriteCommitted 
    LevelRepeatableRead 
    LevelSnapshot 
    LevelSerializable 
    LevelLinearizable 
）
```

# 來試著操作看看DB
## 1. 使用MySQL驅動
```go=1
package main

import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
	"fmt"
)

func main() {
}
```
就這樣...
疑?  別懷疑!!!
我們來看看sql包提供什麼方法給mysql注入驅動
mysql又怎麼偷偷去注入的.

database/sql包的Register(), 有提供該方法注入驅動名稱跟驅動物件實例
```go
func Register(name string, driver driver.Driver)
```

go-sql-driver/mysql則是在driver.go這裡有註冊了init要執行的動作.
```go
// go-sql-driver/mysql/blob/master/driver.go
package mysql

import (
	"database/sql"
	"database/sql/driver"
)

func init() {
	sql.Register("mysql", &MySQLDriver{})
}
```
還記得之前提到的package在相互import時, 如果有init會呼叫吧!
就是在這裡偷偷的注入了驅動.

有了驅動, 就來建立連線.

## 2. 嘗試建立連線
```go=1
package main

import (
	"database/sql"
	"fmt"

	"github.com/go-sql-driver/mysql"
)

func main() {
	config := mysql.Config{
		User:                 "root",
		Passwd:               "m_root_pwd",
		Addr:                 "172.31.0.11:3306",
		Net:                  "tcp",
		DBName:               "testSync",
		AllowNativePasswords: true,
	}

	fmt.Println("conn: ", config.FormatDSN())
	// db, err := sql.Open("mysql", "root:m_root_pwd@tcp(172.31.0.11:3306)/testSync")
	// Open()並不會真的去連接DB
	db, err := sql.Open("mysql", config.FormatDSN())
	if err != nil {
		fmt.Println(err)
	}

	// 釋放連線
	defer db.Close()

	// Ping會真的建立一條連線
	err = db.Ping()
	if err != nil {
		fmt.Println(err)
	}
}
/*
conn:  root:m_root_pwd@tcp(172.31.0.11:3306)/testSync?maxAllowedPacket=0
*/
```
![](https://i.imgur.com/tLlrQ2s.png)
![](https://i.imgur.com/5cFsdaN.png)
利用中斷點可以明顯的知道Open()並無真正建立連線.
DB位置亂打也不會真的給錯誤XD.

這裡的連線字串是採用DSN(Data Source Name)的結構
[username[:password]@][protocol[(host:[port])]]/dbname[?param1=value1&...&paramN=valueN]

config.FormatDSN()則是把config轉成DSN格式的字串

**重要提醒**
DB的連線都是被設計來當作長連線使用的, 所以不該頻繁的Open、Close.
Open()也並不是真正建立連線, 也沒去驗證連線參數, 只是提早準備好資料庫的抽象實例, 方便等等使用.
Open取得的db實例, 要重複一直利用, 不應該去重複生成.
但若程式退出, 最好在主執行緒上執行db.Close().
不然就要等MySQL主動來確認這連線是否還健康.



## 3.執行基本語句
```go=1
package main

import (
	"database/sql"
	"fmt"
	"log"

	"github.com/go-sql-driver/mysql"
)

const (
	CreateUsersTable = "CREATE TABLE IF NOT EXISTS `users` (`user_id` bigint(20) NOT NULL AUTO_INCREMENT, `user_name` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL,	`created_time` timestamp NOT NULL DEFAULT current_timestamp,	`updated_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,	PRIMARY KEY (user_id)  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci"
)

func logIfErr(err error) {
	if err != nil {
		log.Printf("error occurred: %s", err)
	}
}

func main() {
	config := mysql.Config{
		User:                 "root",
		Passwd:               "m_root_pwd",
		Addr:                 "172.31.0.11:3306",
		Net:                  "tcp",
		DBName:               "testSync",
		AllowNativePasswords: true,
	}

	fmt.Println("conn: ", config.FormatDSN())
	// db, err := sql.Open("mysql", "root:m_root_pwd@tcp(172.31.0.11:3306)/testSync")
	db, err := sql.Open("mysql", config.FormatDSN())
	logIfErr(err)
	defer db.Close()
	// create users table
	_, err = db.Exec(CreateUsersTable)
	logIfErr(err)

	// insert data
	result, err := db.Exec("INSERT INTO `users` (`user_name`) VALUES('test02')")
	logIfErr(err)
	rowCount, err := result.RowsAffected()
	logIfErr(err)
	log.Print(rowCount)

	// query data
	rows, err := db.Query("SELECT user_name FROM users")
	logIfErr(err)

	for rows.Next() {
		var s string
		err = rows.Scan(&s)
		logIfErr(err)
		log.Print("%q", s)
	}
	rows.Close()
    
    // query data with arguments
	rows, err = db.Query("SELECT user_name FROM users WHERE user_name = ?", "nathan")
	logIfErr(err)

	for rows.Next() {
		var s string
		err = rows.Scan(&s)
		logIfErr(err)
		log.Print("%q", s)
	}
	rows.Close()
}
```

# 預編譯語句 Prepared Statement
資料庫會接收到各種不同的敘述來執行. 尤其就以查詢來說, 很可能內容都是一樣的, 只是where條件稍微不同. 但是每一次接收到敘述時都還是要 檢查->解析->執行->回傳
這樣的一套流程.
要是我們可以省下檢查->解析的過程, 每次只要把變數代入做 執行->回傳的動作的話. 
速度上會快上一些.
![](https://i.imgur.com/WAN0NuL.png)

```sql
SELECT user_name FROM users WHERE user_name = ? ; 
```
這個?我們叫做佔位符號(placeholders); 用來避免直接在sql作字串拼接, 可以防止大部分的[sql injection](https://www.acunetix.com/websitesecurity/sql-injection/)

|MySQL|Postgres|Oracle|
|:---:|:---:|:---:|
|WHERE col = ?  |WHERE col = $1|WHERE col = :col|
|VALUES(?, ?, ?)|VALUES($1, $2, $3)|VALUES(:val1, :val2, :val3)|

預處理過得語句在資料庫中會被存起來, 資料庫會對該語句先作檢查->剖析, 作執行計畫的判斷跟語句優化.
而後我們只要帶入變數資料, 就能直接執行了.

```go
userName := "nathan"
stmt, err := db.Prepare("SELECT user_name FROM users WHERE user_name = ? ;")
if err != nil {
    log.Fatal(err)
}
rows, err := stmt.Query(userName)
defer stmt.Close()
```
```go
stmt,_ := db.Prepare("SELECT uid,username FROM USER WHERE age = ?")
defer stmt.Close()

// 同樣語句, 不同變數; 這樣連線也只要跟連線池要一次就好
for age := 18; age < 100 ; age ++ {
    rows,_ = stmt.Query(age)
    defer rows.Close()
    for rows.Next(){
         var name string
         var id int
        if err := rows.Scan(&id,&name); err != nil {
            log.Fatal(err)
        }
    }
}
```

```go
stmt,_ := db.Prepare("SELECT uid,username FROM USER WHERE age = ?")
defer stmt.Close()

// 同樣語句, 不同變數; 這樣連線也只要跟連線池要一次就好
	for outerAge := 18; outerAge < 100; outerAge++ {
		go func(age int) {
			rows, _ = stmt.Query(age)
			defer rows.Close()
			for rows.Next() {
				var name string
				var id int
				if err := rows.Scan(&id, &name); err != nil {
					log.Fatal(err)
				}
			}
		}(outerAge)
	}
```

## Query()
- DB.Query(query string, args ...interface{}) (*Rows, error)
- Tx.Query(query string, args ...interface{}) (*Rows, error)
- Stmt.Query(args ...interface{}) (*Rows, error)
前兩個只是參數多了sql查詢語句.
Stmt本來就先保存了一份prepared stmt在資料庫了, 這裡就只是傳遞參數.
回傳值都是一樣的.
使用就全看我們的情境了.

Tx幾乎都是為了先鎖定確認資料的值, 才決定執行異動.
或者是多筆異動都要確保一起完成或失敗.

## Rows.Next()
透過Next()得知還有沒有下一筆資料.
這時就能思考是否這時Query成功後, 其實所有的資料都在Go的服務的記憶中了??

其實不是XD  真要是這樣那些撈報表的早就記憶體被塞爆了.
也沒有必要去手動遞延呼叫rows.Close(), 來歸還連線.

之間的溝通全靠[COM_QUERY](https://dev.mysql.com/doc/internals/en/com-query-response.html#packet-ProtocolText::ResultsetRow)在串流式的發送命令和讀取TCP package進到buffer.
直到收到EOF時, 就表示沒有下一筆了.
[readRow()原始程式](https://github.com/go-sql-driver/mysql/blob/8056f2ca4aa7be2e2e10ab01426a630d5a5bfa81/packets.go)

我以後再補一篇在個人網誌, 用wireshark就能抓到這mysql數據包了

明天會介紹更多DB用法, 應該吧.

> 資料庫許多事情, 能到[BackendTw](https://www.facebook.com/groups/616369245163622/)臉書社團詢問T大, 他的經驗跟講座都很值得聽.

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10220392)