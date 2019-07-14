---
title: Rx簡介
date: 2019-07-14 12:21:13
tags:
    - Rx
---
# When to use Rx

### 使用Rx來精心地安排非同步和事件流的計算
經常為了處理單一事件或是非同步的計算，而把程式的結構搞得非常的複雜，通常會設計狀態機來循序處理。
還得處理每一個節點的成功跟失敗端點。這讓程式非常難以了解跟維護。

Rx的出現，使得這些計算跟處理成為一等公民(First-class citizens)。提供了一些模型和可組合的API來處理這些非同步操作。

Sample :
```csharp
var scheduler = new ControlScheduler(this);
var keyDown = Observale.FromEvent<KeyEventHandler, KeyEventArgs>(
    d => d.Invoke, 
    h => textBox.keyUp += h,
    h => textBox.KeyUp -= h
);
var dictionarySuggest = keyDown.Select( _ =>textBox1.Text)
                               .Where(text =>!string.IsNullOrEmpty(text))
                               .DistingctUntilChanged()
                               .Throttle(TimeSpan.FromMilliseconds(250), scheduler)
                               .SelectMany(
                                   text => AsyncLookupInDictionary(text)
                                            .TakeUntil(keyDown)
                               );
dictionarySuggest.Subscribe(
    results => listView1.Items.AddRange(results.Select(
        result =>new ListViewItem(result)).ToArray()
    ),
    error =>LogError(error)
);
```

這範例展示了UI如何接收用戶的鍵入並接收顯示。
透過Rx建立了一個可觀察的序列(Observable sequence)，依附在KeyUp事件下。
然後每個事件的上層，嵌入了幾個filter和projection， 確保事件只有透過事件觸發時，會發射event stream和唯一的值。
像是KeyUp事件每次都會戳一次，但是其他動作並不會。
並且透過Throttle操作子，確保在250ms區間內的行為只會觸發一次，透過延遲觸發節省昂貴的查找。

在傳統的作法上，Throttling的做法通常是透過timer callback來實作，但是timer本身很可能就會錯誤並拋出exceptions。
一旦用戶鍵入並過濾完畢，就可以執行字典查找了，但通常這會透過Http來做請求，所以這個操作本身就是個async操作。
SelectMany操作子允許輕鬆的組合多個async操作，不只組合了成功的狀態，也能追蹤每個單獨操作中出現的異常。
在以往，這通常是引入不同的callback，
如果用戶在操作時，仍然繼續鍵入新的值，通常會希望不會在看到之前操作的結果，因此舊的查詢結果就不必再顯示出來。
TakeUntil操作確保，一旦偵測到新的KeyDown，就會忽略字典的查找。

最後我們訂閱這個observable sequence的結果，我們掛載了2個函式在訂閱的呼叫上

1. 接收成功的計算結果
2. 接收異常

### 使用Rx開始來處理非同步序列的資料
Rx 遵循著以下幾個文法 OnNext* (OnCompleted|OnError)?。
這些文法允許多個信息隨著時間的推移而倒入，使得Rx適用於能操作單個信息的操作，甚至於多個信息。

Sameple :
```csharp
//open a 4GB file for async reading in block of 64k
var inFile = new FileStream(@"d:\temp\4GBfile.txt", 
    FileMode.Open, 
    FileAccess.Read,
    FileShare.Read,
    2 << 15,
    true);
//open a file for async writing in blocks of 64k
var outFile = new FileStream(@"d:\temp\Encrypted.txt",
    FileMode.OpenOrCreate,
    FileAccess.Write,
    FileShare.None,
    2 << 15, 
    true);
inFile.AsyncRead(2 << 15)
      .Select(Encrypt)
      .WriteToStream(outFile)
      .Subscribe(
          _ =>Console.WriteLine("Successfully encrypted the file."),
          error => Console.WriteLine(
              "An error occurred while encrypting the file :{0},
              error.Message
          )
      );
```

在這範例中，4GB的檔案，被整個讀取，並且透過加密存到另一個檔案。
讀取整份檔案進去記憶體，透過加密跟寫檔出來，這是個非常高成本的操作。
取而代之，我們依靠Rx可以產生許多個信息的event stream。
以64K的區塊來非同步讀取文件，這產生了一個observable sequence。
然後我們分別加密每個區塊，一旦區塊經過加密，就會立即的被發送到下一個管線，已被保存到另一份文件中。
WriteToStream操作就是一個可以處理多個信息的非同步操作。

### The Rx Contract
IObservable和IObserver只用來這些方法的參數和回傳型別。
Rx類別對這兩個介面做了比.net更多的假設。
這些假設使得所有Rx類型的producer和consumbers都應該遵從的行為契約。
這份契約使得去推論和證明程式的正確性。

### Rx的假設文法
信息被送到IObserver介面時必須遵從的文法 :
OnNext* (OnCompleted |OnError)?
這組文法允許observable sequences去送出任意數量的OnNext信息到 被訂閱的observer實例中。
或是單一結果的成功(OnCompleted), 或是任何的失敗(OnError)。

單一信息能很明確的指示出這一個observer sequence的消費者可以安全地執行清理操作。
單一的失敗信息，能確保多個observable sequences可以終止。

Sample :
```csharp
var count = 0;
xs.Subscribe(v => {
    count++ ;
    },
    e => Console.WriteLine(e.Message),
    () =>Console.WriteLine("OnNext has been called {0} times."), count)
;
```
這範例我們能安全的假設一旦呼叫了OnComplete，OnNext中的調用變數不會被改變。

### 假設observer實例可以被當作Rx給序列化呼叫
由於Rx是使用發布-訂閱模式，在.net中是支援multi threadss的，因此不同的信息可能同時到達不同的thread被處理。
如果observale sequenc的消費者就不得不再每個地方來處理這問題，此時程式就需要實行大量的內文管理，來避免併發問題。
這種方式寫的程式非常難以維護，且效能可能很低落。

由於不是所有的observable sequence都有信息是來自不同的執行緒的上下文，
因此只有下述這種obervale sequence的producer才需要做序列化，確保消費者，可以安全的假設信息是以序列化的方式到達。

Sample :
```csharp
var count = 0;
xs.Subscribe(v=>{
    count++;
    Console.WriteLine("OnNext has been called {0} times", count)
});
```
在這範例中，不需要對count做任何lock或是讀寫互斥鎖的實作，因為只有OnNext的呼叫可以ˇ隨時得讀取和寫值到count。

### 確保在OnError和OnCompleted之後，資源會被清除
上面指出了，只要OnError或是OnCompleted被調用後，就不會再有信息被送達。
因此可以確保在OnError或是OnCompleted被觸發後，清除任何訂閱使用的資源。

Sample :
```csharp
Observavle.Using(
    () => new FileStream(@"d:\temp\test.txt", 
        FileMode.Create),
    fs => Observable.Range(0, 10000)
            .Select(v => Encoding.ASCII.GetBytes(v.ToString()))
            .WriteToStream(fs))
          .Subscribe();
```
這範例中使用了Using去建立資源，這資源將會被disposed在unsubscription被呼叫之後。

### 盡最大努力去退訂所有未完成的工作
當unsubscribe被呼叫後，observable sequence將會盡最大的努力去阻止所有未完成的工作。
這也意味著還沒開始的排隊作業都不會被啟用。
任何已經在進行中的工作都可能完成。因為放棄正在進行中的工作並不是一個安全的行為。
只是這些工作的結果並不會被發送到任何以前訂閱的觀察者的實例中了。

Sample 1 :
```csharp
Observable.Timer(TimeSpan.FromSeconds(2)).Subscribe(...)Displose()
```
在這範例中，訂閱由Timer建立出來的oberservable sequence將在ThreadPool scheduler去形成一個排隊列緒，在2秒內去發送OnNext信息。
訂閱之後立即取消，由於排成計畫尚未開始，因此將從scheduler中刪除。

Sample 2 :
```csharp
Observable.Start(() => {
    Thread.Sleep(TimeSpan.FromSeconds(2));
    return 5;
}).Subscribe(...).Dispose();
```
在這範例之中，Start操作子，立即安排lambda function做完參數。訂閱後將observer實例作為此執行的監聽器。
由於一旦訂閱被執行，它將繼續運行並且忽略返回值5。

