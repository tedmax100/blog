---
title: Factory Pattern Cont’d
date: 2019-02-20 13:48:39
tags:
    - Design Pattern
    - TypeScript
    - JavaScript
---
## Simple Factory Method (簡單工廠模式)
也稱為靜態工廠模式，去掉了抽象工廠類別，簡單實現，但缺點 工廠類的擴展就困難了，會不符合開閉原則。
![](/images/DP/P81JUXV.jpg)

```javascript
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
        console.log("黑人會說話，一般人聽不懂");
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
class HumanFactory  {
    public static createHuman<T extends Human>(type: (new () => T)): T {
        let human: any = {};
       //  let testType: new() => T| undefined ;
        debugger;
        try {
             human = new type();//(<any>Object).assign(human,  new type());
        }catch(exp) {
            console.error(exp);
        }
        return human as T;
    }
}
(() => {
    let whiteUhman = HumanFactory.createHuman<WhiteHuman>(WhiteHuman);
    whiteUhman.getColor();
    whiteUhman.talk();
    let balckhman = HumanFactory.createHuman<BlackHuman>(BlackHuman);
    balckhman.getColor();
    balckhman.talk();
    let yellowHuman = HumanFactory.createHuman<YellowHuman>(YellowHuman);
    yellowHuman.getColor();
    yellowHuman.talk();
})();
```

## Multiple Factorys
往往在複雜的業務項目上，會遇到一個產品類，有超多種的實現類。
每個實現類的初始化方法都不太依樣，如果寫在一個工廠方法之中，一定會導致該方法複雜無比。
要讓結構清晰，就替每個產品定義一個創造者，然後由調用者去選擇與哪個工廠方法做關聯。
![](/images/DP/6Zx2ffj.jpg)
好處 創建類別職責清晰，且結構簡單，但是可擴展性和維護帶來一定影響。
因為多一個產品，就要堆一個工廠類，還得考慮對象之間的關係。
```javascript
namespace MultipleFactories {
    
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
        console.log("黑人會說話，一般人聽不懂");
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
abstract class AbstractHumanFactory {
    public abstract createHuman(): Human ;
}
class YellowHumanFactory extends AbstractHumanFactory {
    public createHuman = (): YellowHuman =>  {
        return new YellowHuman();
    }
}
class BlackHumanFactory extends AbstractHumanFactory {
    public createHuman = (): BlackHuman =>  {
        return new BlackHuman();
    }
}
class WhiteHumanFactory extends AbstractHumanFactory {
    public createHuman = (): WhiteHuman =>  {
        return new WhiteHuman();
    }
}
(() => {
    let whiteUhman = new WhiteHumanFactory().createHuman();
    whiteUhman.getColor();
    whiteUhman.talk();
    let balckhman = new BlackHumanFactory().createHuman();
    balckhman.getColor();
    balckhman.talk();
    let yellowHuman = new YellowHumanFactory().createHuman();
    yellowHuman.getColor();
    yellowHuman.talk();
})();
}
```

## Lazy initialization 延遲初始化
一個物件被消費完成後，不立刻釋放，而是保持其初始狀態，等待被再度使用。
這是工廠模式的一種擴展應用。
![](/images/DP/K0UJ66B.jpg)
```javascript
class Product {}
class ConcreteProduct1 extends Product {}
class ConcreteProduct2 extends Product{}
class ProductFactory {
    private static prMap: Map<string, Product> = new Map();
    public static createProduct: Product = (type: string): Product|undefined => {
        let product = null;
        if(ProductFactory.prMap.has(type)) {
            product = ProductFactory.prMap.get(type);
        }else{
            if(type === "Product1") {
                product = new ConcreteProduct1();
            }else{
                product = new ConcreteProduct2();
            }
            // 把物件 放到緩存中
            ProductFactory.prMap.set(type, product);
        }
        return product;
    }
}
```

舉例，像是Conneection Pool都會要求設置MaxConnection最大連線數量，該數量就是記憶體中instance的數量。

## Conclusion
很多官方與第三方套件之中都包含工廠方法，且工廠方法還能與其他模式混搭使用(模板模式、單例、原型模式等)，有多更適合的設計。