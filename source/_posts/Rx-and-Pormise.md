---
title: Rx and Pormise
date: 2019-02-14 12:59:23
tags:
    - Rx
    - JavaScript
---
# Reactive Programing (響應式編程)
Def : 一種面向(data flow)數據流和(propagation of change)變化傳播的編程風格。

* propagation of change 變化傳播
最初的資料是否會隨著後續對應變量的變化而變化。
    * 在inperative programming中
A+B=C
*2+3=5
2+4=5 not 6
當B的資料發生改變之後，C的數值必沒有隨著B的改變而改變。
    * 在Reactive Programing中
A+B=C
2+3=5
2+4=6
    * 在MVVM中，存在一種M到V的綁定關係
![](/images/Rx/ofzzCyr.png)
當Model由model1變為model2時，View也隨之進行了變化，從view1變成view2.
所以MVVM框架，也實現了RX中的propagation of change概念。

* data flow(stream)
監聽一系列的事件流，並對這一系列事件進行 映射(Map)、過濾(Filter)、合併(Merge)等處理後，在響應整個事件流的callback(回調)，該過程便是 面相數據流的編程。

數據流被封裝在一個叫做Observable的實例中，通過觀察者模式，對數據流進行統一的訂閱，並在中間插入像filter這樣的操作，從而對Observable所封裝的數據流進行處理。

```
myObservable.filter(fn).subscribe(callback);
```

### ReactiveX
微軟開發維護的Reactive Programing套件。
結合了 觀察者模式、迭代器模式、函數式編程(Functional Programming)。

* Observable
Rx核心概念!!
所有產生出來的非同步數據都先包裝程Observable對象，Observable對象是把這些非同步數據轉換成 data stream的形式。所以這些Observable對象等同於data stream的源頭，後續操作都圍繞著這些被轉換的流動數據展開的。
![](/images/Rx/fXMFsde.png)
上圖最上方的箭頭(時間軸)表示了最初的Observable對象，這個對象發出了3個數據，這3個可能是點擊事件的數據，也可能是response的數據。
經過map處理後，原來的Observable對象會變成一個新的Observable對象，並且原來的3個數據會轉換成新的數據在新的Observable對象數據流裡流動。

概念雷同於 工廠生產線上的 生產流水線。
![](/images/Rx/Img244356699.jpg)
Observable對象相當於半成品，map相當於流水線上的工人，加工後變成成品。

* RX 是 借鑒了集合的操作思想，把複雜的非同步數據流處理問題，簡化成同步的集合處理問題。
換言之，開發者能透過Observable，操作集合一樣操作複雜的非同步數據流。
* Operator
Operator是實現了迭代器模式、函數式編程的利器。
Operator是Observable的操作方式。每一個數據流，都能透過某個operator對該Observable對象進行操作。大部分operator操作完後，會返回一個新的Observable對象給下一個operator處理。
也因此，這樣方便在各個operator間透過鍊式寫法編寫。

```
let newObservable = observable
                    .debounceTime(500)
                    .take(2);
```

### Compare With Promise
能用promise的場景, RxJs都適用，因為RxJs是作為promise的超集合存在的。
```javascript
let promise = new Promise((resolve, reject) => {
    // some code
    if(/* 異步執行成功 */) resolve(value);
    reject(error);
});
```
```javascript
let observable = new Observable((observer) => {
    observer.next(value1);
    observer.next(value2);
    
    observer.error(err);
})
```
Pormise只能針對單一的非同步事件進行resolve()，但在Observable中，不僅能處理單一的非同步事件(就是調用observer的next())，而且能以streaming形式響應多個非同步事件。
還有對於Promise中的all()、race()等，RxJs都有對應的解決方案。

```javascript
let newPromise = Promise.all(promiseReq1, promiseReq2);
let newObservable = Rx.Observable.forkJoin(obsReq1, obsReq2);
```