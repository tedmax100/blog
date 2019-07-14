---
title: JS Clean Code訓練營
date: 2018-10-09 12:46:57
tags:
    - JavaScript
    - CleanCode
---
![](/images/Refactor/51ta2ZRmPeL.jpg)
```
第一天 :
有效的单元测试
识别依赖
隔离依赖
前端逻辑的常见剥离方式
Stub与Mock
处理Callback和Promise
第二天 : 
小步重构
识别代码臭味
处理代码臭味的技巧
良好设计的基本原则
消除重复，降低复杂度
```

## Day1 :
### Lesson 1:
1 .FizzBuzz

數學歸納法: n =1 => n+1 ok
一個it test, 通常證明2個test case

Vue_Header profileCaption test
### Lesson 2:
單元測試的好處:
* “提早”得知程式碼是否有漏洞; 提早設置check point。
* 回歸測試
* 自動運行
* 說明文件
* 建立可重複利用的元件
* 任何測試案例應該是獨立的
* 使用者角度去做測試案例
* private method不應該被特別測試，因為第7點的關係，使用者只在意公開方法，且私有方法一定會被公開方法給使用到。
* 基本上單元測試，都是黑盒測試，因為只在意輸出入。
* 單元測試要快
* 使用類似jest的框架，整合模擬測試ui上的行為

### Lesson 3 :
物件導向設計原則:

* 組件 應該要具備 高內聚、低耦合 的特性
* 物件導向的繼承關係，就是高耦合
* 只要function內有new()，也是高耦合

Stub : state change
Mock : behavior test (called, parameter…)

### Lesson 4:
一一一一一一一一一一一一一一一一一一一
UT —————————–> IT

* Unit Test:
    * 速度快
    * 程式少
    * 定位問題簡單
    * 代碼依賴剝離 成本高
    * 無網路、文檔讀寫、外部第三方套件、跟運行環境無關的、與配置無關

* Integration Test:
    * 速度慢
    * 程式多
    * 定位問題多
    * 代碼依賴剝離 成本低
    * 前端的UI
    * 異步執行

#### Refactoring
* moment.js(會修改自身，產生副作用) -> date.js

* budget.js
    * if 有return , else也return
    * remove else
    * 重複的邏輯 -> extract method
    * 可讀性低 || 註解 (因為怕看不懂，所以加註解)

* code smell
    * 長方法 long method代碼行數過長 (code standard by group define)
    * 多種數據結構 使用同一個數據 表達同一件事情。startDate & endDate 進來後被轉成兩種不同的變數。
    * duplicate logic(code)
    * temporary variable(Field) -> Inline Temp
    * 令人費解的命名 (不把型別加入命名中) -> *Rename method
    * clearly intention
    * 資料謎團 data clump
    * 抽象干擾 Abstraction Distraction
    * 特性忌妒 Feature Envy
    * 基礎型別偏執 primitive obssession

* how
    * use inline option
    * delete useless codes
    * delete duplicate codes ; 先讓疑似重複的部分盡可能變得一樣，別一開始就提取代碼。
    * extract method
    * rename
    * covert param to object
    * create data class
    * change signature
    * use loadash to make code to be clearly intention

#### Refactor vs Rewrite
* Refactor: 行為與之前一樣，但代碼可讀性更高
* Rewrite : 行為未必與之前一樣。
* 異動範圍大小
* 流程與結構的不同

## Day2
被提取出來的私有class，與私有方法，只要是只有被測試公開方法給涵蓋，且只有這些再用，就不必再額外寫測試。除非它後來有其他其他未被涵蓋測試的公開方法給引用。除非後來因為這些被提取的部分出bug，再補充其測試。

* Refactor
    * 擁抱變化
    * 從legacy code 實現 演進式設計
* 建築 vs software design
    * 藍圖 –build–> 建築
    * 代碼 –build–> 軟體
    * UML 用來與人溝通，一致化想法用的語言，並非藍圖
* JS 不一定適合套用design pattern, 只有少數的pattern適用

* Legacy code type :
    * new feature
    * stable (已經上無數補丁，正在運行中)
    * unused

### Lesson 2
* MVC vs MVP
    * MVC的進入點是controller -> Model -> Controller -> View
    * MVP的進入點式VIEW -> Presenter -> Model -> Presenter -> View
* Tell, Don’t ask -> Design principle

* 防衛性編程
```
fun A(xxx) {
    if(A) ...
    return ___;
}
let a = A(param);
if(a) {
    ....
}
```

* 在進入口做防範驗證，內部邏輯別做太多返回值的判斷。
* 調用者保證參數有效，被調用者保證返回值有效
* Design Contract
* 給予 default value; 別用null 表達某一種邏輯
* Null object pattern
* architecture
* view model
* presenter
* business model
* dto、dao


### Smells
* 註解
    * 不適當的訊息 Inappropriate Information : 註釋只應該描述有關代碼跟設計的技術性訊息，不該帶作者、最後修改時間等。 因為GIT上會記錄。
    * 廢棄的註解 Obsolete Comment:不正確或無關的註解。
    * 冗餘註解
    * 糟糕的註解
    * 註解掉的代碼  

* 方法
    * 過多的參數 Too Many Arguments : 盡量少，沒參數最好，超過3個就要避免。
    * 輸出參數 Output Arguments: appendFooter(s) 不如把footer設定在物件屬性內，再呼叫report.appendFoorer()
    * 標示參數 Flag Arguments

```
//pseudo-code
class Concert...
  public Booking book (Customer aCustomer, boolean isPremium) {...}
```
```
public Booking regularBook(Customer aCustomer) {...}
public Booking premiumBook(Customer aCustomer) {...}
```

    * 死方法 Dead Function

* 一般問題
    * 重複 DRY (Don’t Repeat Yourself): 資料庫正規化，物件導向繼承
    * 接口提供過多 Too Much Information
    * 特性依戀 Feature Envy: 類別的方法只對類中的屬性跟方法有興趣，不該依靠其他類中的變數跟方法。顯然是「內聚力」不夠的一種現象