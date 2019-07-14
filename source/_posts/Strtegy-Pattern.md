---
title: Strtegy Pattern
date: 2019-02-23 13:57:02
tags:
    - Design Pattern
    - TypeScript
    - JavaScript
---
### 小故事
劉備去東吳招親前，諸葛亮預測東吳會刁難劉備，甚至吞掉荊州西川，因此諸葛亮特授予趙雲三個錦囊，說是按照天機拆開解決棘手問題。

三個妙計是:
* 找喬國老幫忙(走後門)
* 求吳國太放行(訴苦)
* 孫夫人斷後(親情攻擊)  

這三個妙計都是告訴照雲要怎麼去執行，也就是說三個計謀都有一個方法是”執行“。
具體執行什麼內容，每個妙計會有所不同。

類別圖 :
![](/images/DP/xZEF4gP.png)
```javascript
interface IStrategy {
    // 執行錦囊
    operate(): void;
}
class BackDoor implements IStrategy {
    public operate = () => console.log("找喬國老幫忙，讓吳國太施予壓力");
}
class GivenGreenLight implements IStrategy {
    public operate = () => console.log("找吳國太開綠燈，給予放行");
}
class BlockEnemy implements IStrategy {
    public operate = () => console.log("孫夫人斷後，擋住追兵");
}
```

還需要裝著計策的錦囊，以及一個執行人 趙雲。
![](/images/DP/yFCmo8g.png)
```javascript
class Context {
    private strategy: IStrategy;
    constructor(strategy: IStrategy) {
        this.strategy = strategy;
    } 
    public operate = () => this.strategy.operate();
}
// ZhaoYun
(() => {
    let context: Context;
    console.log("--剛到吳國拆第一個--");
    context = new Context(new BackDoor());
    context.operate();
    console.log("------------------");
    console.log("--劉備樂不思蜀，拆第二個--");
    context = new Context(new GivenGreenLight());
    context.operate();
    console.log("------------------");
    console.log("--孫權小兵殺來，拆第三個--");
    context = new Context(new BlockEnemy());
    context.operate();
    console.log("------------------");
})()
```

## 策略模式的定義
Define a family of algorithms, encapsulate each one, and make them interchangeable.
(定義一組算法，將每個算法封裝起來，並且使它們之間可以互換。)
![](/images/DP/jC5J5vp.png)
```javascript
interface IStrategy {
    doSomething(): void;
}
class ConcreteStrategy1 implements IStrategy {
    public doSomething = () => console.log(1);
}
class ConcreteStrategy2 implements IStrategy {
    public doSomething = () => console.log(2);
}
class Context {
    private strategy: IStrategy;
    
    constructor(_strategy: IStrategy) {
        this.strategy = _strategy;
    }
    public doAnything = () => this.strategy.doSomething();
}
(() => {
   let strategy = new ConcreteStrategy1();
   let context: Context = new Context(strategy);
   context.doAnything();
})()
```
### 回顧  
* 使用了 OO的繼承跟多態。
* 要定義哪個行為，是抽象策略介面。
### Pros :  
* 算法可以自由切換 : 只要有實現抽象策略，就成為了策略家族的一個成員，通過封裝腳色對其進行封裝，保證對外提供”可自由切換”的策略。
* 去除多重條件判斷 : 因為多重條件不容易維護閱讀，且改壞的機率很大。使用了策略模式後，可能由其他模塊決定採用什麼策略，對外提供的訪問接口就是封裝類別，簡化了操作，也避免掉條件判斷。
* 擴展性良好 : 只要新增一個策略成員並且實現接口就能, 類似於一個可反覆拆裝的插件，為此這模式符合了OCP原則。
### Cons :  
* 策略類別太多 = 類別膨脹, 可重複利用程度非常小。
* 所有策略類別都需要對外暴露: 上層模組一定要知道有哪些策略，才能決定怎麼使用。這樣違反了LKP原則(最少知識原則); 但可以用工廠模式、代理模式或是享元模式解決。

### 使用場景
* 多個類別只有在算法或是行為上稍有不同的場景
* 算法需要自由切換的場景
* 需要屏蔽算法規則的場景 ; 調用者不必了解太多細節，能依據策略名稱就能知道怎使用，然後反饋給他一個極果，就結束了。
* 如果策略超過4個，應該要考慮混和模式，解決策略類別過長跟對外暴露的問題。


### 策略模式的例子
#### 加減乘除計算器####
**Version 1**
```javascript
class Calculator {
    private static readonly ADD_SYMBOL: string = "+";
    private static readonly SUB_SYMBOL: string = "-";
    public exec = (a: number, b: number, symbol: string) => {
        let result: number = 0;
        if(symbol === Calculator.ADD_SYMBOL){
            result = this.add(a, b);
        }else if (symbol === Calculator.SUB_SYMBOL) {
            result = this.sub(a, b);
        }
        return result;
    }
    private add = (a: number, b: number) => a + b;
    private sub = (a: number, b: number) => a - b;
}
(() => {
    let a = 1;
    let symbol:string = "+";
    let b = 2;
    let cal = new Calculator();
    console.log(cal.exec(a, b, symbol));
})()
```

**Version 2**
使用三元運算子簡化主邏輯
```javascript

2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
class Calculator {
    private static readonly ADD_SYMBOL: string = "+";
    private static readonly SUB_SYMBOL: string = "-";
    public exec = (a: number, b: number, symbol: string) => 
        symbol === Calculator.ADD_SYMBOL ? a+b : a-b;
    
}
(() => {
    let a = 1;
    let symbol:string = "+";
    let b = 2;
    let cal = new Calculator();
    console.log(cal.exec(a, b, symbol));
})()
```

**Version 3 引入策略模式**
由上下文角色決定具體策略; 並且封裝角色保證策略時可以相互替換
```javascript
interface Calculator {
    exec(a: number, b: number): number;
}
class Add implements Calculator {
    constructor() {}
    public exec = (a: number, b: number): number => a+b; 
}
class Sub implements Calculator {
    constructor() {}
    public exec = (a: number, b: number): number => a-b; 
}
class Context {
    private cal: Calculator;
    constructor(_cal: Calculator) {
        this.cal = _cal;
    }
    public exec = (a: number, b: number, symbol: string) => this.cal.exec(a,b);
}
(() => {
    const ADD_SYMBOL: string = "+";
    const SUB_SYMBOL: string = "-";
    let a = 1;
    let symbol:string = "+";
    let b = 2;
    let context: Context;
    if (symbol === ADD_SYMBOL) context = new Context(new Add());
    else if (symbol === SUB_SYMBOL) context = new Context(new Sub());
    console.log(context!.exec(a, b, symbol));
})()
```

**Version 4 引入策略列舉**