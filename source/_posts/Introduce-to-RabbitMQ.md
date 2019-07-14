---
title: Introduce to RabbitMQ
date: 2017-12-13 12:29:48
tags:
    - MQ
    - RabbitMQ
---
![](/images/MQ/RabbitMQRouting.png)
[RabbitMQ Tutorials
](https://www.rabbitmq.com/getstarted.html)
<!--more-->
### What is RabbitMQ?
RabbitMQ是實現AMQP的一種微服務，用於分散是系統之中來儲存轉發訊息，便於使用，方便擴展，又有高可用性。
目的能替系統之間做雙向解耦。當生產者產出大量資料要送出時，消費者若無法快速消費掉，這時候就需要一個中介層，來保存這些數據。

AMQP的工作流程如下圖 : message被publisher 發送給exchange，exchange常常被比喻為郵局或是郵箱。然後exchange根據收到的message以及規則分發給綁定的queue。最後AMQP代理會將message投遞給訂閱此queue的consumer，或是消費者依照需求自行獲取。
![](/images/MQ/exchanges-topic-fanout-direct.png)

因為網路是不可靠的，接收消息的服務也有可能在處理時失敗，所以AMQP包含的一了message acknowledgement的概念:當一個message從queue當中投遞給consumer後，consumer會通知broker，這個message可以從queue當中刪除。

某些情況下，當message無法被成功投遞時，message或許會被返回給producer並且被丟棄。或者代理執行了延期操作，message會被放入Dead-Letter exchange中。此時producer可選選擇某些參數來處理這些特殊情況。

一個Log系統，能用MQ來簡化工作，一個consumer進行訊息的正常處理，另一個consumer對訊息做log紀錄，只要在系統中，起兩個consumer並把queue以相同的方式binding到同一個exchange即可。 剩下的訊息分派工作全由MQ負責完成。

### Concept and Feature
* Broker
就是MQ service本身

* Producer
![](/images/MQ/producer.webp)
發送message的程序

* Consumer
![](/images/MQ/consumer.png)
一個等待從queue當中獲取message的程序

* Virtual Host
一個broker內可以設置多個Vhost，做為不同用戶的權限分離，或是不同的業務規劃。
Vhost之間相互隔離，不同Vhost之間無法共享exchange/queue。

* Exchange
![](/images/MQ/exchanges.webp)
交換機，指定消息按照什麼規則，路由到哪個queue。
有direct、topic、headers、fanout 四種type能設置,
不同type的exchange路由行為是不同的。

* Queue
![](/images/MQ/queue.webp)
每個message都會被投遞到一個或是多個queue當中，等待被投遞。
類似於郵筒的概念，message都會被存放在此。
queue本身是一個很大的message buffer，可以有很多個producer發送，
但都會傳到同一個queue且可以有多個consumer獲取資料。

* Channel
在客戶端的每個connection中，可以建立多個channel,每個channel表示一個session， 客戶端只能透過channel才能執行AMQP的命令。
之所以需要channel因為TCP連線的建立跟釋放都是十分昂貴的，如果一個客戶端的每個線程都需要與broker交換訊息，每一個線程都建立一個TCP connection的話，OS也無法承受每秒建立如此多的TCP connection。所以RabbitMQ建議同一個發送串行資料的線程共用Channel和connection。
* Binding
把exchange和queue按照路由規則綁釘起來。
Exchange在跟多個queue binding後會生成一張routing table，
* Routing Key
路由的關鍵字，exchange根據這關鍵字，來進行訊息投遞
* AMQP entities
Queue + Exchange + Binding = AMQP entities

### Task Queues
Task queues工作隊列，是為了避免等待一些占用大量資源、費時的操作。只要把task當作訊息丟進queue中，就會有運行的worker取出任務然後處理，當運行多個workers時，任務就會在彼此之間分配。

#### Message Acknowledgment
通常沒特別設置ack在queue的時候，只要message一投遞出去，立刻就為從queue之中移除。
此時如果worker運行到一半掛掉，正在處理的message就會遺失了。
如果不想遺失任何message，當前worker掛掉時，我們希望任務會重新指派給其他worker。

因此，RabbitMQ提供了acknowledements。
worker會通過一個ack訊號，告訴RabbitMQ已經收到了並且處理完該條訊息，然後MQ就會刪除該訊息。

如果worker掛了，沒有發送ack，則MQ就會認為message沒有被完全處理，就會重新發送給其他worker，這樣就不會遺失任何message。

但由於message沒有timeout的概念，只能等worker跟MQ斷開連線，這樣MQ就會重送了。
message acknowledement預設是**關閉**的，只要把auto_ack = true即可。

```csharp
var consumer = new EventingBasicConsumer(channel);
consumer.Receved += (model, ea) =>{
    var body = ea.body;
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine($"{received {message}}");
    int dots = message.Split('.').Length - 1;
    Thread.Sleep(dots * 1000);
    Console.WriteLine("Done");
    channel.BasicAck(deliveryTag : ea.DeliveryTag, multiple:false);
};
channel.BasicAck(queue:"task_queue", autoAck:True, consumer :consumer);

```

**Increase throuput and performance**  
關閉ack能提升MQ的效能

**Message Durability**
Message Durability訊息持久化，
MQ預設並不會對queue和message做持久化的設置，因此必須先把queue和message設置為durable。

首先先聲明queue為durable，這樣確保MQ重啟後，queue不會被遺失。
MQ也不允許使用不同參數定義一個同名的queue。因此producer和consumer的設置必須一樣。

```csharp
channel.QueueDeclare(queue:"task_queue",
                     durable :true,
                     exclusive:false,
                     autoDelete:false,
                     arguments:null);
```
接著設置message persistence
```csharp
var properties = channel.CreateBasicProperties();
properties.Persistent = true;
```
*Note on message persistence*  
把message 設置為persistence，並不能完全保證不會丟失。因為只是告訴MQ要把message存入硬碟，MQ也不是所有message都寫入硬碟，可能只是放在記憶體暫存。

**Fair Dispatch**
MQ只管把第n-th消息投遞給第n-th個worker，並不關心worker有沒有ack。
可以設置prefetch_count=1，告訴MQ，同一時間，別送超過1條訊息給同一位worker，直到他已經處理完上一條message並送出ack。這樣MQ就會把消息分發給下一位worker。
```csharp
channel.basicqos(0, 1, false);
```

### Publish/Subscribe
pub/sub目的是要把一個message分發給多個consumer。
這個模型的核心概念是，producer並不會直接發送訊息給queue，而是把消息發送給exchange。

exchange在這裡就是負責從producer接收消息，一邊把消息推送到queue。
exchange必須知道如何處理它接受到的消息，是要推送到指定的queue還是多個queue，或是忽略，這些規則是透過exchange type來定義。
![](/images/MQ/exchanges.webp)

**Exchange Type**
* direct 直連
* fanout 廣播
* topic 主題
* headers 表頭
```csharp	
channel.ExchangeDeclare("logs", "fanout");
```

*Note for default exchange*
MQ預設就存在一組default exchagne，名稱是空字串””

**Temporary queues**
在task queue的情境下，給個worker同樣的queue name，這時候會透過round robin做輪詢派送。
但是現在要的是每個人都能收到同樣的訊息，因此需要的是一個全新、空的queue，來跟exchagne綁定。
能透過自己定義隨機的queue name或是，讓MQ來幫我們選擇一個隨機的queue name。

```csharp
var queueName = channel.QueueDeclare().QueueName;
```
這時拿到的queue name就會類似amq.gen-JzTY20BRgKO-HjmUJj0wLg。

**Bindings**
![](/images/MQ/bindings.webp)
有了fanout exchange和數個queue，這時就要設置exchange和queue之間的關聯。

```csharp
channel.QueueBind(queue:queueName,
                 exchagne:"logs",
                 routingKey:"");

```
![](/images/MQ/python-three-overall.png)

*Note for routing key*
fanout type下，routing key是會被忽略的。

### Routing
Routing key的設置，能使得queue只訂閱消息的子集合。
綁定的時候可以設置routingKey又或是稱為bindingKey，為了避免跟BasicPublish的routingKey搞混。

```csharp
channel.QueueBind(queue:queueName, 
                  exchange:"direct_logs",
                  routingKey:"black");
```
使用exchange和routing key來進行精確配對，從而確保消息該投遞到哪個queue。

![](/images/MQ/direct-exchange.png)
這裡第一個queue用orange作為綁定鍵，另一個queue用black和green。
這樣所有orange的消息都會被路由到C1, 而black/green則會被路由到C2，其他message通通被丟棄。

**Multiple Bindings**
多個queue使用同樣的routingKey也是可行的。

![](/images/MQ/direct-exchange-multiple.webp)

C1和C2都使用black做綁定。這樣跟fanout type的行為雷同，只要是black的訊息，C1跟C2都會收到，但其他一樣被丟棄。

**Scene : Log System**
將log依據不同級別作為rougingKey來選擇接收者跟處理方式。
[](/images/MQ/python-four.webp)
建立exchange
```csharp
channel.ExchangeDeclare(exchange: "direct_logs", type: "direct");
```
發送log訊息, serverity是info、warning、error其中一個
```csharp
var body = Encoding.UTF8.GetBytes(message);
channel.BasicPublish(exchange: "direct_logs",
                     routingKey: severity,
                     basicProperties: null,
                     body: body);
```

subscribing
```csharp
var queueName = channel.QueueDeclare().QueueName;
foreach(var severity in args)
{
    channel.QueueBind(queue: queueName,
                      exchange: "direct_logs",
                      routingKey: severity);
}
```

### Topics
Direct exchange有些限制，沒辦法基於多個標準來執行路由操作。
有時會希望不只是訂閱基於嚴重程度的日誌，也希望訂閱其他種日誌。
這時就需要topic exchange。

發送到topic exchange的訊息不可以設置routingKey，它的routingKey是一個由.分隔開的單字列表。這些單字是什麼都能，跟message有關係的詞彙是最好的。
例如:”stock.usd.nyse”, “quick.orange.rabbit”，單字個數可以任意個，但不能超過255 bytes。
routingKey中也能使用類似regular expression表達個數:

* * 表示一個單字
* # 表示任意數量的單字
[](/images/MQ/python-five.webp)
一個攜帶有quick.orange.rabbit的消息會被投遞到C1跟C2。
攜帶著lazy.orange.elephant的也是。
quick.orange,fox只會投遞給C2。
lazy.pink.rabbit只會給C2投遞1次。
quick.brown.fox的將會被丟棄。
orange和quick.orange.male.rabbit的都會被丟棄掉。
lazy.orange.male.rabbit將會被投遞到C2。

`topic exchange是很powerful的，它可以表現出其他exchnge type的行為。
當一個queue的routingKey是#時，這個queue將會無視message的routingKey，接收全部message。
當*和#都未出現在routingKey時，這時候就跟direct type是一樣的行為。`

**Questions**
* bindingKey為*的queue會取到一個routingKey為空字串的消息嗎?
* bindingKey為#.的queue會收到一個routingKey為*..的消息嗎? 它會收到routingKey為一個單字的消息嗎?
* a.*.#和a.#的區別?

### RPC
遠端過程調用Remote Procedure Call(RPC)
如果需要將一個函式運行在遠端服務上並且等待結果時，這時就需要RPC。

透過RabbitMQ來建造一個RPC System，一個client和一個RPC Server。
[](/images/MQ/python-six.webp)

**client interface**
```csharp
var rpcClient = new RPCClient();
Console.WriteLine("Request fib()");
var response = rpcClient.Call("30");
Console.WriteLine($"Got {0}", response);
```

### RabbitMQ Cluster and High Available
RabbitMQ Cluser設置要求
所有機器上的Erlang和RabbitMQ版本需要都相同，機器上Erlang的Cookie也相同
