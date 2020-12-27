---
title: Go_Reflection
date: 2020-12-21 21:36:09
categories: "Go"
tags:
    - Go
    - iT邦鐵人賽11Th
---
![](https://ct.yimg.com/xd/api/res/1.2/5B7x8F4s6a.YL9BnXGWAiA--/YXBwaWQ9eXR3YXVjdGlvbnNlcnZpY2U7aD00NTg7cT04NTtyb3RhdGU9YXV0bzt3PTcwMA--/https://s.yimg.com/ob/image/d19578ac-9012-4614-8b72-5594dc44b3fc.jpg)

# Reflection 反射
反射指的是程式"運行"期間動態的調用對象的方法和屬性.
Golang內建這功能, 在"reflect"包裡.
​<!-- more -->
思考一下interface{}這類型, 什麼都能代入.
但這麼方便的話, 我們幹麻不全傳它就好XD
Go在這裡面做了很多反射取型別的動作判斷.
不然幹麻出type assertion(類型斷言)這類的東西.
就是要想辦法告訴Go這變數是什麼類型, 要怎使用它.


# Type & Value & Kind
一個變數一定包含 type和value兩個部份.
理解這點就知道為什麼 nil != nil, 像是字串的nil跟int的nil就不會一樣.

## [Kind](https://golang.org/pkg/reflect/#Kind)
對象隸屬的種類...
```go
const (
    Invalid Kind = iota
    Bool  
    Int
    ...
    Array
    Chan
    Func
    Interface
    Map
    Ptr
    Slice
    String
    Struct
    UnsafePointer
)
```
很多...基本上就粗分成
`基礎型別`, `引用型別`, `指針`這樣

## [Type](https://golang.org/pkg/reflect/#Type)
Type代表的是對象的類別資訊

- TypeOf  透過TypeOf取得對象的Type資訊
```go
func TypeOf(i interface{}) Type
```
- Elem 透過Elem取得對象包含的值或者是對象是個指針所指向的值
```go=1
package main

import (
	"fmt"
	"reflect"
)

type Enum int

const (
	Zero Enum = 0
)

func main() {
	type cat struct {
	}

	// 實例化物件
	typeOfCat := reflect.TypeOf(cat{})
	fmt.Println(typeOfCat.Name(), typeOfCat.Kind())

	// 基礎型別
	typeOfA := reflect.TypeOf(Zero)
	fmt.Println(typeOfA.Name(), typeOfA.Kind())

	// 指針類型, 指針是沒有類別名稱的, 不是nil是空字串
	catInstance := &cat{}
	typeOfCatInstance := reflect.TypeOf(catInstance)
	fmt.Printf("name: '%v' kind:'%v'\n", typeOfCatInstance.Name(), typeOfCatInstance.Kind())

	//  透過Elem()來取得這指針指向的對象
	typeOfCatInstanceElem := typeOfCatInstance.Elem()
	fmt.Printf("name: '%v' kind:'%v'\n", typeOfCatInstanceElem.Name(), typeOfCatInstanceElem.Kind())
}

/*
cat struct
Enum int
name: '' kind:'ptr'
name: 'cat' kind:'struct'
*/
```

## [Value](https://golang.org/pkg/reflect/#Value)
Value就對象的原始值
```go=1
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var a int = 1024
	// 透過valueof 取得對象的值
	valueOfA := reflect.ValueOf(a)

	// 將對象的值以interface{} 類型返回取得
	// 再做類型斷言
	var getA int = valueOfA.Interface().(int)

	// 將對象的值以int 類型返回取得
	var getA2 int = int(valueOfA.Int())

	fmt.Println(getA, getA2)
}
// 1024 1024
```

## 獲得struct的成員訊息
```go=1
package main

import (
	"fmt"
	"reflect"
)

// struct
type Dummy struct {
	a int
	b string
	float32
	bool
	// 指針成員
	next   *Dummy
	nilPtr *Dummy
}

func main() {
	// dummy實例
	dummyIns := Dummy{
		next: &Dummy{},
	}
	// 取得成員屬性數量
	d := reflect.ValueOf(dummyIns)
	fmt.Println("NumField:", d.NumField())

	// 透過索引來取得成員屬性
	floatField := d.Field(2)
	fmt.Println("Field2 :", floatField.Type())

	// 透過屬性名稱來取得成員屬性
	fmt.Println("FiledByName:(\"b\".Type", d.FieldByName("b").Type())

	// 根據[]int的的值當索引取得成員屬性數量
	// []int{4, 0}, 4先取得的是next; 0指向的就是最初的成員 a int
	fmt.Println("FieldByIndex([]ubt{4,0}).Type()", d.FieldByIndex([]int{4, 0}).Type())

	// IsNil()判斷該成員是不是nil
	fmt.Println("nilPtr *Dummy IsNill()", d.FieldByName("nilPtr").IsNil())

	// IsValid()判斷該成員的值是否合法, 就是看是否有值, 或是值是否有效
	// nilPtr 是個指針本身有值, 只是指向nil
	fmt.Println("nilPtr *Dummy IsValid()", d.FieldByName("nilPtr").IsValid())
	// 查找abc()
	fmt.Println("func abc()", d.MethodByName("abc").IsValid())

	// 這裡故意存取一個不存在的成員
	fmt.Println("notExist property IsValid()", d.FieldByName("").IsValid())
	// nil
	fmt.Println("nil IsValid()", reflect.ValueOf(nil).IsValid())

	// 從map中查找一個不存在的鍵
	mapIns := make(map[string]int)
	fmt.Println("not exist key of map IsValid()", reflect.ValueOf(mapIns).MapIndex(reflect.ValueOf("aa")).IsValid())
}

/*
NumField: 6
Field2 : float32
FiledByName:("b".Type string
FieldByIndex([]ubt{4,0}).Type() int
nilPtr *Dummy IsNill() true
nilPtr *Dummy IsValid() true
func abc() false
notExist property IsValid() false
nil IsValid() false
not exist key of map IsValid() false
*/
```

## Struct Tag 結構體標籤
我們在JSON、SQL、ORM處理序列化或反序列化時, 都需要用到這些tag.
用tag來設定該屬性在處理時應該具備的特殊屬性或行為.

tag格式: Key跟Value用:分隔, 值用"包起來; 不同鍵用一個空格作分隔.
```
`Key1:"Value1" Key2:"Value"`
```
```go=1
package main

import (
	"encoding/json"
	"fmt"
	"reflect"
)

type Cat struct {
	Name   string  `json:"name"`
	Type   int     `json:"type" id:"100"`
	Money  float64 `json:"money"`
	Ignore bool    `json:"-"` // - 表示會被忽略,  直接賦予該類型的零值
}

func main() {
	ins := Cat{
		Name: "black",
		Type: 1,
	}

	typeOfCat := reflect.TypeOf(ins)

	for i := 0; i < typeOfCat.NumField(); i++ {
		fieldType := typeOfCat.Field(i)

		fmt.Printf("name: %v ,tag:'%v'\n", fieldType.Name, fieldType.Tag)
	}

	if catType, ok := typeOfCat.FieldByName("Type"); ok {
		fmt.Println(catType.Tag.Get("json"))
		fmt.Println(catType.Tag.Get("id"))
	}

	// 透過Marshal(), 來把資料轉成json字串, 這裡會去讀取json tag
	j, err := json.Marshal(ins)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(string(j))

	// 透過Unmarshal(), 來把json字串轉成對應物件, 這裡會去讀取json tag
	var jsonBody = []byte(`{"name":"yellow","type":2,"money":200}`)
	var cat Cat
	err = json.Unmarshal(jsonBody, &cat)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Printf("%+v", cat)
}

/*
name: Name ,tag:'json:"name"'
name: Type ,tag:'json:"type" id:"100"'
name: Money ,tag:'json:"money"'
name: Ignore ,tag:'json:"-"'
type
100
{"name":"black","type":1,"money":0}
{Name:yellow Type:2 Money:200 Ignore:false}
*/
```


### Interface
之前提過實現Interface是透過隱性的向上轉型的方式(Duck typing).
來稍微用反射玩看看, 當然實際不是這樣做的, 效能會大打折扣.
Reflect相當吃效能.
```go=1
package main

import (
	"fmt"
	"reflect"
)

type CatPtrInterface interface {
	MeowPtrReceiver()
}

type CatInterface interface {
	MeowReceiver()
}

type Cat struct{}

func (c Cat) MeowReceiver() {
	fmt.Println("meow")
}

func (c *Cat) MeowPtrReceiver() {
	fmt.Println("meowptr")
}

type DogPtrInterface interface {
	Woof()
}

func main() {
	catPtrObject := &Cat{}
	//catPtrInterfaceType := reflect.TypeOf((*CatPtrInterface)(nil)).Elem()
	catPtrInterfaceType := reflect.TypeOf(new(CatPtrInterface)).Elem()
	catPtrObjectType := reflect.TypeOf(catPtrObject)
	fmt.Println("&Cat{} implement CatPtrInterface? ", catPtrObjectType.Implements(catPtrInterfaceType))

	catObject := Cat{}
    // 建立一個CatInterface指標nil
	catInterfaceType := reflect.TypeOf((*CatInterface)(nil)).Elem()
	catObjectType := reflect.TypeOf(catObject)
	fmt.Println("Cat{} implement CatInterface? ", catObjectType.Implements(catInterfaceType))

	dogInterfaceType := reflect.TypeOf((*DogPtrInterface)(nil)).Elem()
	fmt.Println("&Cat{} implement DogPtrInterface? ", catPtrObjectType.Implements(dogInterfaceType))
	fmt.Println("Cat{} implement DogPtrInterface? ", catObjectType.Implements(dogInterfaceType))
}
/*
&Cat{} implement CatPtrInterface?  true
Cat{} implement CatInterface?  true
&Cat{} implement DogPtrInterface?  false
Cat{} implement DogPtrInterface?  false
*/
```

> 題外話
> 小弟最近把Go升級到1.13版...結果VSCode沒法執行Debug  QQ
> 我的作法是把/usr/loca/go的資料刪除, 重新安裝GO1.12版 

[鐵人賽連結](https://ithelp.ithome.com.tw/articles/10219894)