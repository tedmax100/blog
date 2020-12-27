---
title: Go database/sql Scan & Value, 讓操作sql有一點點ORM的感覺
date: 2020-12-21 21:38:10
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
# Scanner & Valuer
​<!-- more -->
```go
// package "database/sql"
type Scanner interface {
    Scan(src interface{}) error
}
// package "database/sql/driver"
type Valuer interface {
    Value() (Value, error)
}
```
Scanner的Scan()讀取從資料庫傳來的內容，並轉成符合自己的格式；
也就是說Rows或者Row的Scan()其實就是調用每個來源類型的Scan(), 將其存到來源變數上, 來源變數必須滿足driver.Value的類型.

相對的，Valuer 則是把自己的資料結構，轉成sql看得懂的形式。
也就是把Go的類型轉成driver.Value的對應類型.

建立一張user_tbl表
```sql
CREATE TABLE `user_tbl` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `userName` varchar(100) CHARACTER SET utf8 NOT NULL,
  `nickName` varchar(40) CHARACTER SET utf8 DEFAULT NULL,
  `createTime` bigint(20) DEFAULT NULL,
  `registTime` datetime DEFAULT NULL,
  `alive` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_user_tbl_userName` (`userName`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

把[昨天提到的部分](https://ithelp.ithome.com.tw/articles/10220392), 寫一下來簡單的執行.
```go=1
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"reflect"
	"time"

	"github.com/go-sql-driver/mysql"
)

type YesOrNo bool

const (
	Yes YesOrNo = true
	No          = false
)

type UserTbl struct {
	Id         int       `db:"id"`
	UserName   string    `db:"userName"`
	NickName   string    `db:"nickName"`
	CreateTime int64     `db:"createTime"`
	RegistTime time.Time `db:"registTime"`
	Alive      YesOrNo   `db:"alive"`
	prvate     int
}

func NewEmptyUserTbl() UserTbl {
	return UserTbl{}
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

	var err error
	fmt.Println("conn: ", config.FormatDSN())
	// db, err := sql.Open("mysql", "root:m_root_pwd@tcp(172.31.0.11:3306)/testSync")
	// Open()並不會真的去連接DB
	db, err := sql.Open("mysql", config.FormatDSN())
	// 連線池中最大空閒連線數量
	db.SetMaxIdleConns(10)
	// 連接中的最大數量
	db.SetMaxOpenConns(2)
	// 連線可以被重用的最大存活時間
	db.SetConnMaxLifetime(time.Second * 600)

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

	usertbl := &UserTbl{
		UserName:   "Nathan-1",
		NickName:   "Thor-1",
		CreateTime: 1569420293000,
		RegistTime: time.Now(),
		Alive:      Yes,
	}

	ctx, cancelCb := context.WithCancel(context.Background())

	insertResult, _ := db.ExecContext(ctx, "INSERT INTO user_tbl  (userName, nickName, createTime, registTime, alive) VALUES(?, ?, ?,?, ?)",
		usertbl.UserName, usertbl.NickName, usertbl.CreateTime, usertbl.RegistTime, usertbl.Alive)


	userResults := make([]UserTbl, 0)
	rows, err := db.QueryContext(ctx, "SELECT  nickName, userName, createTime, registTime, alive FROM user_tbl")
	for rows.Next() {
		usertbl := NewEmptyUserTbl()
	
		rows.Scan(&usertbl.UserName, &usertbl.NickName, &usertbl.CreateTime, &usertbl.RegistTime, &usertbl.Alive)
		userResults = append(userResults, usertbl)
	}
    // 歸還連線
	rows.Close()
    for idx := range userResults {
		fmt.Println(userResults[idx])
	}

	cancelCb()
}
/*
{0 Thor Nathan 1569420293000 0001-01-01 00:00:00 +0000 UTC false 0}
{0 Thor-1 Nathan-1 1569420293000 0001-01-01 00:00:00 +0000 UTC false 0}
*/
```
一般用法, 有多少欄位, 就要在scan列舉出所有相對物件的成員屬性, 不美觀;  未來也要改很多地方的程式.

透過reflect, 把值反射進去對應名稱的成員
```go=1
func GetData(rows *sql.Rows, dest interface{}) error {
	// 取得資料的每一列的名稱
	col_names, err := rows.Columns()
	if err != nil {
		return err
	}
	// 取得變數對象的值跟類型資訊
	v := reflect.ValueOf(dest)
	if v.Elem().Type().Kind() != reflect.Struct {
		return errors.New("give me  a struct")
	}
	// 宣告一個interface{}的slice
	scan_dest := []interface{}{}
	// 建立一個string, interface{}的map
	addr_by_col_name := map[string]interface{}{}

	for i := 0; i < v.Elem().NumField(); i++ {
		propertyName := v.Elem().Field(i)
		col_name := v.Elem().Type().Field(i).Tag.Get("db")
		if col_name == "" {
			if v.Elem().Field(i).CanInterface() == false {
				continue
			}
			col_name = propertyName.Type().Name()
		}
		// Addr() 返回該屬性的記憶體位置的指針
		// Interface() 返回該屬性真正的值, 這裡還是存著位置
		addr_by_col_name[col_name] = propertyName.Addr().Interface()
	}
	// 把實際各成員屬性的位置, 給加到scan_dest中
	for _, col_name := range col_names {
		scan_dest = append(scan_dest, addr_by_col_name[col_name])
	}
	// 執行Scan
	return rows.Scan(scan_dest...)
}
```
這樣使用舒服多了.
但應該發現Alive這怎樣都是false.
不是資料庫存錯, 是Go這時候不認得怎樣Scan這種YesOrNo類型.


```go
	for rows.Next() {
		usertbl := NewEmptyUserTbl()
		// 一般用法, 有多少欄位, 就要在scan列舉出所有相對物件的成員屬性, 不美觀
		// rows.Scan(&usertbl.UserName, &usertbl.NickName, &usertbl.CreateTime, &usertbl.RegistTime, &usertbl.Alive)
        
        // 直接給rows跟對應的結構體指針
		GetData(rows, &usertbl)
		userResults = append(userResults, usertbl)
	}
```

# [Driver](https://golang.org/pkg/database/sql/driver/)
這裡面定義很多接口, 
其中有各種類型的ValueConverter接口的實現. 
用途有
- 互相轉換Go原生資料類型到MySql的資料類型
- 轉換row的值, 變成driver.Value類型
- Scan()將driver.Value類型轉成用戶定義的類型

```go=1
func (yon YesOrNo) Value() (driver.Value, error) {
	return bool(yon), nil
}

func (yon *YesOrNo) Scan(src interface{}) error {
    // row裡面存的資料是空, 就給預設值
	if src == nil {
		*yon = YesOrNo(false)
	}
	// row裡面存的資料轉成支援的driver value類型
	if bv, err := driver.Bool.ConvertValue(src); err == nil {
		// 如果driver.Value能斷言成bool成功的話
		if v, ok := bv.(bool); ok {
			// 賦值給yon
			*yon = YesOrNo(v)
			return nil
		}
	}
	// 無法轉成支援的driver value, 就噴錯
	return errors.New("scan fail for YesOrNo")
}
/*
{0 Thor Nathan 1569420293000 0001-01-01 00:00:00 +0000 UTC false 0}
{0 Thor-1 Nathan-1 1569420293000 0001-01-01 00:00:00 +0000 UTC true 0}
*/
```
能正常顯示了!


## Null Value
我改成設定registTime, 但這個欄位我是允許NULL, 且我真的沒特別設定usertbl.RegistTime, 所以它是零值.
```go
usertbl := &UserTbl{
		UserName:   "Nathan-2",
		NickName:   "Thor-2",
		CreateTime: 1569420293000,
		// RegistTime: time.Now(),
		Alive: Yes,
}
_, err = db.ExecContext(ctx, "INSERT INTO user_tbl  (userName, nickName, createTime, registTime, alive) VALUES(?, ?, ?,?, ?)",
		usertbl.UserName, usertbl.NickName, usertbl.CreateTime, usertbl.RegistTime, usertbl.Alive)
if err != nil {
    fmt.Println(err)
}  
// Error 1292: Incorrect datetime value: '0000-00-00' for column 'registTime' at row 1
```

之前提到的組合就能用了
```go
// 自定義一個Time結構, 內嵌time.Time
type Time struct {
	time.Time

	valid bool
}
// 實作Value接口
func (t Time) Value() (driver.Value, error) {
    // 當t的Time是零值時, 返回nil這值
	if t.IsZero() {
		return nil, nil
	}
	return t.Time, nil
}

func (t *Time) Scan(src interface{}) error {
	if src == nil {
		t.Time, t.valid = time.Time{}, false
		return nil
	}

	if t.Time, t.valid = src.(time.Time); t.valid {
		return nil
	}

	return errors.New("scan fail for Time")
}
```
![](https://i.imgur.com/a19yzfd.png)
這時就能看到有幾筆資料的registTime就會是NULL了.
Scan()也是如此. 我們都得實作這些Null的特殊處理

先把mysql.Config中的ParseTime設定成true, 這幫助我們處理NullTime
```go
config := mysql.Config{
    User:                 "root",
    Passwd:               "m_root_pwd",
    Addr:                 "172.31.0.11:3306",
    Net:                  "tcp",
    DBName:               "testSync",
    AllowNativePasswords: true,
    ParseTime:            true,
}
```
執行看看, 改成有撈取registTime
```go
userResults := make([]UserTbl, 0)
rows, err := db.QueryContext(ctx, "SELECT  nickName, userName, registTime, alive FROM user_tbl")
for rows.Next() {
    usertbl := NewEmptyUserTbl()
    GetData(rows, &usertbl)
    userResults = append(userResults, usertbl)
}
/*
{0 Nathan Thor  0 0001-01-01 00:00:00 +0000 UTC false false 0}
{0 Nathan-1 Thor-1  0 2019-09-25 15:55:41 +0000 UTC true false 0}
{0 Nathan-2 Thor-2  0 2019-09-25 17:14:25 +0000 UTC true false 0}
{0 Nathan-3 Thor-3  0 0001-01-01 00:00:00 +0000 UTC true false 0}
{0 Nathan-4 Thor-4  0 0001-01-01 00:00:00 +0000 UTC true false 0}
*/
```

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10220925)