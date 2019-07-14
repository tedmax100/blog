---
title: Decorator_Pattern
date: 2019-04-23 14:09:19
tags:
    - Design Pattern
    - TypeScript
    - JavaScript
---
做武器系統
今天想模仿暗黑破壞神的武器系統那樣
![](/images/DP/xqVWxxO.png)

利用前綴詞為武器加上能力。
[D2魔法前綴詞表](http://wiki.d.163.com/index.php?title=Magic_Prefixes_and_Suffixes_(Diablo2)

首先我先建立一個基礎類別，然後各種武器(刀、劍、斧 等等)繼承於它。
```javascript
export abstract class BaseWeapon {
    private name: string;
    private attackPower: number;
    constructor(name: string, attackPower: number) {
        this.name = name;
        this.attackPower = attackPower;
    }
    public Name = (): string => this.name;
    public AttackPower = (): number => this.attackPower;
}
```


再來建立了一個劍和匕首類別
```javascript
import { BaseWeapon } from "./BaseWeapon";
export  class Sword extends BaseWeapon {
    constructor() {
        super("Sword", 9);
    }
    public Name = (): string => super.Name();
    public  AttackPower = (): number => super.AttackPower();
}
export  class Gull extends BaseWeapon {
    constructor() {
        super("Gull", 2);
    }
    public Name = (): string => super.Name();
    public  AttackPower = (): number => super.AttackPower();
}
```

然後透過繼承生成出了Flery Gull、Flery Sword、Static Gull、Static Sword
```javascript
import { Gull } from "./Gull";
import { Sword } from "./Sword";
export  class FleryGull extends Gull {
    constructor()  {
        super()
    }
    public Name = (): string => `烈焰的${super.Name}`;
    public  AttackPower = (): number => super.AttackPower() + 16;
}
export  class FlerySword extends Sword {
    constructor()  {
        super()
    }
    public Name = (): string => `烈焰的${super.Name}`;
    public  AttackPower = (): number => super.AttackPower() + 16;
}
export  class StaticGull extends Gull {
    constructor()  {
        super()
    }
    public Name = (): string => `靜電的${super.Name}`;
    public  AttackPower = (): number => super.AttackPower() + 4;
}
export  class StaticSword extends Sword {
    constructor()  {
        super()
    }
    public Name = (): string => `靜電的${super.Name}`;
    public  AttackPower = (): number => super.AttackPower() + 4;
}
```
```javascript
let sword = new Sword();
console.log(`${sword.Name} : ${sword.AttackPower}`);
let gull = new Gull();
console.log(`${gull.Name} : ${gull.AttackPower}`);
let flerySword = new FlerySword();
console.log(`${flerySword.Name} : ${flerySword.AttackPower}`);
let staicGull = new StaticGull();
console.log(`${staicGull.Name} : ${staicGull.AttackPower}`);
```
```
Sword : 9
Gull : 2
烈焰的Sword : 25
靜電的Gull : 6
```

Class Diagram
![](/images/DP/aDtcx6P.png)

**But!!!**
這才2種武器，2個特效，我已經有4個類別(2*2)。 2層繼承。
當我前綴又再一層時，或者有後綴的出現，整個很難維護。
再這情境上，我很可能會有一把是”靜電的烈焰”或”烈焰的靜電”
這樣在現在設計上是不同類別，太多本質相似的類別需要維護了。

整理發生幾個現象 : **繼承層數過多**、**類別數量激增**
繼承超過兩層，可以想想是不是自己設計上出了問題，因為這樣維護成本只會越來越繁重。

為解決這些問題，增加一個抽象方法或介面類別來封裝武器類別。
從上面可發現，我第二層跟第三層的類別基本上都是有共同的介面去實作一些行為。

Components: (要被裝飾物件)
```javascript
export interface BaseWeapon {
    Name(): string;
    AttackPower(): number;
    Attack(): void;
}
export  class Sword implements BaseWeapon {
    private name: string;
    private attackPower: number;
    constructor() {
        this.name = "Sword";
        this.attackPower = 9;
    }
    public Name = (): string => this.name;
    public  AttackPower = (): number => this.attackPower;
    public Attack = (): void => console.log(`${this.name}打出了${this.attackPower}點傷害!`);
}
export  class Gull implements BaseWeapon {
    private name: string;
    private attackPower: number;
    constructor() {
        this.name = "Gull";
        this.attackPower = 2;
    }
    public Name = (): string => this.name;
    public  AttackPower = (): number => this.attackPower;
    public Attack = (): void => console.log(`${this.name}打出了${this.attackPower}點傷害!`);
}
```

Decorators : 要加上的動態職責，需要有跟Components一樣的介面。
```javascript
export abstract class WeaponDecorator implements BaseWeapon{
    protected name: string;
    protected attackPower: number;
    protected weapon: BaseWeapon;
    constructor(name: string, attackPower: number, weapon: BaseWeapon) {
        this.name = name;
        this.attackPower = attackPower;
        this.weapon = weapon;
    }
    public Name = (): string => this.name + this.weapon.Name();
    public AttackPower = (): number => this.attackPower + this.weapon.AttackPower();
    public Attack = (): void => console.log(`${this.Name()}打出了${this.AttackPower()}點傷害!`);
}
export class FleryDecorator extends WeaponDecorator{
    constructor(weapon: BaseWeapon) {
        debugger;
        if(weapon.Name().indexOf("烈焰的") === -1)
            super("烈焰的", 16, weapon);
        else  super("", 0, weapon);
    }
}
export class StaticDecorator extends WeaponDecorator{
    constructor(weapon: BaseWeapon) {
        if(weapon.Name().indexOf("靜電的") === -1)
            super("靜電的", 4, weapon);
        else super("", 0, weapon);
    }
}
```

```javascript
const sword = new Sword();
    const flerySword = new FleryDecorator(sword);
    flerySword.Attack();
    const staticFleryword = new StaticDecorator(flerySword);
    staticFleryword.Attack();
    const gull = new Gull();
    const staticGull = new StaticDecorator(gull);
    staticGull.Attack();
    const fleryStaticGull = new FleryDecorator(staticGull);
    fleryStaticGull.Attack();
```
```
烈焰的Sword打出了25點傷害!
靜電的烈焰的Sword打出了29點傷害!
靜電的Gull打出了6點傷害!
烈焰的靜電的Gull打出了22點傷害!
```

Class Diagram :
![](/images/DP/ZwtEQJ4.png)
可以看到繼承關係被簡化了，組件跟功能之間變成組合關係。

## 裝飾者模式的定義
Attach additional responsibilities to an object **dynamically** keeping **the same interface**.
Decorators provide a flexible alternative to subclassing for extending functionality.
(動態的給一個對象添加一些額外的職責，就功能面來說，裝是者模式 比 增加子類別靈活)

Compoent抽象類別(BaseWeapon): 原有類別的抽象類或是一組介面
ConcreteCompoent : 被裝飾的具体對象，需要去實現Compoent。
Decorator : 也是一個抽象類別，實現Compontet，且裡面一定要有一個變數指向Componet抽象物件實體。 舉例: WeaponDecorator中的protected weapon: BaseWeapon 。
ConcreteDecorator : Decorator的實作。主要就把基本的東西裝飾成其他東西。

### Pros :
* ConcreteDecorator跟ConcreteComponent可以獨立發展，而不會相互耦合。In other words, 兩方不需要知道彼此的存在。Decorator類是從外部來擴展Component類別的功能，而Decorator也不知道具體的物件。舉例: Decorator依賴的其實是Compoenent的抽象或介面，且是組合關係。
* 是繼承關係的一種替代方案。看Decorator，不管裝飾多少層，返回的對象還是Component的抽象。實現的是is-a的關係。
* 可以動態的擴展一個類別的功能，就是該模式的定義。
* 裝飾者可以擴充Component的狀態，或是修改原有實作方法。  

### Cons :
* 除錯比較困難，多層裝飾下，因為會像是剝洋蔥一樣，可能要撥到最裡面那層，才發現出了問題。  

###使用場景
* 需要擴展一個類別的功能，又或者需要付加給它時。
* 需要動態的給一個對象增加功能，或者是動態的撤回。
* 需要為一大群兄弟類別進行改裝或加裝功能時。

## 使用Curry的概念來練習
```javascript
class Sword {
    private name: string;
    private attackPower: number;
    constructor() {
        this.name = "Sword";
        this.attackPower = 9;
    }
    public Name = (): string => this.name;
    public AddPrefixAndAttackPower = (prefix: string, power: number) => {
        this.name = prefix + this.name; 
        this.attackPower += power;
    }
    public AttackPower = (): number => this.attackPower;
    public Attack = (): void => console.log(`${this.name}打出了${this.attackPower}點傷害!`);
}
interface PrefixValue {
    prefix: string,
    attachPower: number
}
const Prefix = (prefixValue: PrefixValue) => {
    return function(sword: Sword) {
        sword.AddPrefixAndAttackPower(prefixValue.prefix, prefixValue.attachPower);
        return sword;
    }
}
const FleryValue: PrefixValue = {prefix:"烈焰的", attachPower: 16};
Prefix(FleryValue)(new Sword()).Attack();
```
```
烈焰的Sword打出了25點傷害!
```

### 與Proxy的差異?
等Proxy pattern寫完文章，再來一起比較。

### ES6的Decorator?
todo XD