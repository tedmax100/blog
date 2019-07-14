---
title: Rx_IEnumberbale和IObservable
date: 2017-11-26 12:25:25
tags:
    - Rx
---
# Rx可以做的事情
Rx可以處理很多內容，例如async處理、event、IEnumberable等，
例如 :MouseClick event、MousePosition等等的事情
還有對於時間的處理，例如Timer，可以想成是指定時間間隔會發生的事件的值。
還有async，可以想像成某個時間點，才開始進行的處理，且處理完成後，才會得到某個值。
![](/images/Rx/1333383604_7864.gif)


### Rx和Linq
Rx最基本的介面是IObservable(T) (被觀察者)，它與.NET常見的IEnumerable是不同的。
但是能跟用Linq查找的方法一樣來查找。
```csharp
using System.Reactive.Linq;
//LinQ to Objects
var ix = from x in Enumerable.Range(1,10)
         where x % 2 == 0
         select x * x;
var rx = from x in Observable.Range(1,10)
          where x % 2 == 0
          select x * x;
```
Rx裡雖然增加了不少新的方法，但是大部分的同名方法操作跟定義上都跟原本的一樣，這降低了很多學習成本。
雖然介面不同，但是都可以用Linq expression來達成同樣的效果。
所以IObservable/IObserver 和 IEnumerable/IEnumerator，可以視前者為後者的反轉。

這段Code將描述IObserver介面如何反轉IEnumberator介面的。

```csharp
//簡化過的IEnumerator<T>
public interface IEnumberator<T>
{
    T current{get;}
    bool MoveNext();
    //void Reset(); Reset現在一般不使用
}
//MoveNext改回傳bool,再調用current
public interface IEnumberator<T>{
  //MoveNext回傳T的instance
  //如果結束的話，則不回傳(== void)
  //異常的話，拋出
  //因此有3種類型的回傳
  T|void|Excecption GetNext(void);
}
//根據對偶性(Duality)，將參數和回傳值互換位置
//以前都是被動式的去取Pull，現在則是主動的拿到Push，
//所以改用Got
public interface IEnumeratorDual<T>{
    void GotNext(T|void|Exception);
}
//進而按照Pull的3種回傳類型，分開定義介面
public interface IEnumeratorDual<T>{
    void GotNext(T);
    void GotVoid(void);
    void GotException(Exception);
}
//最後視現在用的IObserver<T>介面
public interface IObserver<T>{
    void OnNext(T value);
    void OnComplete();
    void OnError(Exception error);
}
```
所以IObservable和IEnumberator視可以相互轉換的，兩人可透過彼此的擴充方法相互轉換。
且這些擴充方法，在Rx中都已經定義了。
通過這突，可以清楚的了解IEnumerable是Pull,而IObservable是Push。
![](/images/Rx/1334422958_4462.gif)

### Event use Rx
```csharp
//監聽Mouse move event
public static IObservable<MouseEventArgs> MouseMoveAsObservable(this Form form){
    return Observable.FromEventPattern<MouseEventArgs>(from, "MouseMode").Select(e =>e.EventArgs);
}
public void TextChangeAsObservable(){
    //等待1秒後若沒在收到新的資料，就用最近收到的資料來處理
    Observable.FromEventPattern<EventArgs>(textBox, "TextChanged").Select(_ =>textBox.Text)
            .Throttle(TimeSpane.FromSeconds(1));
}
```
Throttle可以設定一定的時間間隔，過濾掉一些不必要的輸入，上面範例中，一秒內無論發生多少次變化，只有最後一次的值才會被push出去。

### Async By Rx
```csharp
var req = WebRequest.Create("http://hoge/");  
req.BeginGetResponse(ar =>  
{  
  try  
  {  
    var res = req.EndGetResponse(ar);  
    var url = new StreamReader(res.GetResponseStream()).ReadToEnd();  
    var req2 = WebRequest.Create(url); // 在前面請求的結果上，再發請請求 
    req2.BeginGetResponse(ar2 =>  
    {  
      //再多次請求的處理下，往往需要在每一層加上try-catch
      try  
      {  
        var res2 = req2.EndGetResponse(ar2);  
        var str = new StreamReader(res2.GetResponseStream()).ReadToEnd();  
        Dispatcher.BeginInvoke(new Action(() => MessageBox.Show(str)));  
      }  
      catch (WebException e) { Dispatcher.BeginInvoke(new Action(() => MessageBox.Show(e.ToString()))); }  
    }, null);  
  }  
  catch (WebException e)  
  {  
    Dispatcher.BeginInvoke(new Action(() => MessageBox.Show(e.ToString())));  
  }  
}, null);
```

```csharp
WebRequest.Create("http://hoge/")
    .DownloadStringAsysnc()
    .SelectMany(url => WebRequest.Create(url).DownloadStringAsync())
    .ObserveOnDisaptcher()
    .Subscribe(
        str =>MeesageBox.Show(str),
        e => MessageBox.Show(e.ToString());
    )
public static class WebRequestExtensions{
    return Observable.FromAsyncPattern<WebResponse>(request.BeginGetResponse, request.EndGetResponse)()
            .Select(res => {
                using(var stram = res.GetResponseStream())
                using(var sr = new StreamReader(stream)){
                    return sr.ReadToEnd();
                }
            });
}
```
使用Rx能把Lambda expression的call back改寫成Method Chain，降低閱讀複雜度。