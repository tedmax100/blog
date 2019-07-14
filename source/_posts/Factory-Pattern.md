---
title: Factory Pattern
date: 2019-02-19 13:45:41
tags:
    - Design Pattern
    - TypeScript
    - JavaScript
---
![](/images/DP/1g0WOw3.jpg)
```javascript
/* 定義人類與人種 */
interface Human {
    //　取得膚色
    getColor(): void;
    //說話
    talk() : void;
}
class BlackHuman implements Human{
    public getColor = () => {
        console.log("黑人膚色是黑色的。");
    }
    public talk = () => {
        console.log("黑人說話，一搬人聽不懂");
    }
}
class YellowHuman implements Human{
    public getColor = () => {
        console.log("黃種人膚色是黃色的。");
    }
    public talk = () => {
        console.log("黃種人會說話，說的都是2byte的文字");
    }
}
class WhiteHuman implements Human{
    public getColor = () => {
        console.log("白人膚色是白色的。");
    }
    public talk = () => {
        console.log("白人會說話，說的都是1byte的文字");
    }
}
```
```javascript
/* 抽象人類工廠，透過泛型對createHuman的輸入參數產生限制 : 
   1. class型別 ; 2. 必須實現Human  */
abstract class AbstractHumanFactory {
    public abstract createHuman<T extends Human>(type: (new () => T)): T ;
}
// 實踐工廠
class HumanFactory extends AbstractHumanFactory {
    public createHuman<T extends Human>(type: (new () => T)): T {
        let human: any = {};
        try {
             human = new type();//(<any>Object).assign(human,  new type());
        }catch(exp) {
            console.error(exp);
        }
        return human as T;
    }
}
```
```javascript
執行　創物者
(() => {
    let creator: AbstractHumanFactory = new HumanFactory();
    let whiteUhman = creator.createHuman<WhiteHuman>(WhiteHuman);
    whiteUhman.getColor();
    whiteUhman.talk();
    let balckhman = creator.createHuman<BlackHuman>(BlackHuman);
    balckhman.getColor();
    balckhman.talk();
    let yellowHuman = creator.createHuman<YellowHuman>(YellowHuman);
    yellowHuman.getColor();
    yellowHuman.talk();
})();
```
```
白人膚色是白色的。
白人會說話，說的都是1byte的文字
黑人膚色是黑色的。
黑人會說話，一般人聽不懂
黃種人膚色是黃色的。
黃種人會說話，說的都是2byte的文字
```

### Definition :
Define an interface for creating an object, but let subclasses decide which class to instantiate Factroy Method lets a class defer instantiation to subclasses.
定義一個用於創建對象的介面，讓子類別決定實例化哪一個類別，使一個類別的實例化延遲到其子類別

### Pros :
* 良好的封裝性，結構清晰，一個對象的建立是有條件約束的，降低耦合性。
* 拓展性優秀，增加業務產品類別十，只要擴展一個工廠類，就能”擁抱變化”。
* 封裝屏蔽產品類，產品類的實現如何變化，調用者根本不需要關心，他只關心產品的接口。
* 滿足 迪米特法則(最小知識原則), 滿足依賴倒置原則，滿足里氏替換原則。
* 萬物皆對象，所以萬物也就是產品類。

### Example :
* JDBC, 從MySQL切換到Oracle，就是更換一下驅動名稱。
* MailServer 有 POP3、IMAP、HTTP，把這三種定義為產品類，定義介面IConnectMail，再定義工廠方法，按照不同條件，選擇不同連接方式，做到完美的拓展。
* 單元測試, 測試類別A，類別有關連到類別B，用工廠方法把類別B虛擬出來，就能Mock依賴物件。