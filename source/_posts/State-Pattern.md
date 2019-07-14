---
title: State Pattern
date: 2017-11-21 13:51:52
tags:
    - Design Pattern
    - TypeScript
    - JavaScript
---
實作電梯
電梯的動作: 開門、關門、運行、停止
![](/images/DP/Nkwi064.jpg)
```javascript
interface ILift {
    open(): void;
    close(): void;
    run(): void;
    stop(): void
}
class Lift implements ILift {
    public open = () => console.log("電梯門打開");
    public close = () => console.log("電梯門關閉");
    public run = () => console.log("電梯上下運行");
    public stop = () => console.log("電梯停止");
}
```
```javascript
(() =>{
    const lift: ILift = new Lift();
    lift.open();
    lift.close();
    lift.run();
    lift.stop();
})()
電梯門打開
電梯門關閉
電梯上下運行
電梯停止
```

**But!!!**
電梯是有狀態的, 有前提條件的。 不可能在運行時突然開門，或是停止了不開門的情況。
所以動作執行都有前置條件，也就是在特定狀態下才能做特定事務。
| | Open | Close | Run | Stop |
|———-|——|——-|—–|——|
| 開門狀態 | x | o | x | x |
| 關門狀態 | o | x | o | o |
| 運行狀態 | x | x | x | o |
| 停止狀態 | o | x | o | x |

### Version 1 : 加上前置條件
```javascript
enum LiftState {
    OPENING_STATE = 1,
    CLOSING_STATE= 2,
    RUNNIG_STATE = 3,
    STOPPING_STATE = 4
}
interface ILift {
    setState(state: LiftState): void;
    open(): void;
    close(): void;
    run(): void;
    stop(): void
}
class Lift implements ILift {
    private state: LiftState;
    constructor() {
        this.state = LiftState.STOPPING_STATE;
    }
    public setState = (value: LiftState) => this.state = value;
    public open = () => {
        switch (this.state) {
            case LiftState.OPENING_STATE :
                break;
            case LiftState.CLOSING_STATE :
                this.openWithoutLogic();
                this.setState(LiftState.OPENING_STATE);
                break;
            case LiftState.RUNNIG_STATE : 
                break;
            case LiftState.STOPPING_STATE :
                this.openWithoutLogic();
                this.setState(LiftState.OPENING_STATE);
                break;
        }
    }
    public close = () => {
        switch (this.state) {
            case LiftState.OPENING_STATE :
                this.closeWithoutLogic();
                this.setState(LiftState.CLOSING_STATE);
                break;
            case LiftState.CLOSING_STATE :
                break;
            case LiftState.RUNNIG_STATE : 
                break;
            case LiftState.STOPPING_STATE :
                break;
        }
    };
    public run = () => {
        switch (this.state) {
            case LiftState.OPENING_STATE :
                break;
            case LiftState.CLOSING_STATE :
                this.runWithoutLogic();
                this.setState(LiftState.RUNNIG_STATE);
                break;
            case LiftState.RUNNIG_STATE : 
                break;
            case LiftState.STOPPING_STATE :
            this.runWithoutLogic();
                this.setState(LiftState.RUNNIG_STATE);
                break;
        }
    }
    public stop = () => {
        switch (this.state) {
            case LiftState.OPENING_STATE :
                break;
            case LiftState.CLOSING_STATE :
                this.stopWithoutLogic();
                this.setState(LiftState.STOPPING_STATE);
                break;
            case LiftState.RUNNIG_STATE : 
                this.stopWithoutLogic();
                this.setState(LiftState.STOPPING_STATE);
                break;
            case LiftState.STOPPING_STATE :
                break;
        }
    }
    private closeWithoutLogic = () => console.log("電梯門關閉...");
    private openWithoutLogic = () => console.log("電梯門開啟...");
    private runWithoutLogic = () => console.log("電梯上下運行...");
    private stopWithoutLogic = () => console.log("電梯停止了...");
}
(() =>{
    const lift: ILift = new Lift();
    lift.open();
    lift.close();
    lift.run();
    lift.stop();
})()
電梯門開啟...
電梯門關閉...
電梯上下運行...
電梯停止了...
```

### 思考問題
實現類別Lift邏輯很饒舌，充斥著很多switch或是if-else。難以閱讀維護。
擴展性很差，當狀態越多(通電狀態、斷電狀態)，都要增加條件。
非常規狀態難以實踐，故障、檢修等狀態。 會違反單一職責原則。
### 轉換思考角度
剛剛都是以電梯的方法跟方法執行的條件去分析。
現在換個角度思考，電梯在具有這些狀態時能夠做什麼事情， 也就是說電梯處於某個具體狀態時，思考這個狀態是由什麼動作觸發而產生的，以及在這狀態下電梯還能做什麼事情?

停止狀態怎來的? 當然是因為執行了stop()
停止狀態下，還能做什麼? 運行? 開門?
所以只要實現電梯在一個狀態下的兩個任務模型即可 :

這個狀態如何產生的
這個狀態下還能做什麼(怎過度狀態)

![](/images/DP/dRe0ZjY.jpg)


### Version 2 : 抽象與撥離
Context
```javascript
import { LiftState } from './state';
export class Context {
    private liftState?: LiftState;
    constructor() {
        this.liftState = undefined;
    }
    public getLiftState = () => this.liftState!;
    public setLiftState = (liftState: LiftState) => {
        this.liftState = liftState;
        this.liftState.setContext(this);
    }
    public open = () => this.liftState!.open();
    public close = () => this.liftState!.close();
    public run = () => this.liftState!.run();
    public stop = () => this.liftState!.stop();
}
```

LiftState
```javascript
import { Context } from "./Context";
let context = new Context();
export abstract class LiftState {
    public setContext = (_context: Context) => context = _context;
    public abstract open(): void;
    public abstract close(): void;
    public abstract run(): void;
    public abstract stop(): void;
}
export class OpenningState extends LiftState {
    constructor() {
        super();
    }
    public close = () => {
        context.setLiftState(new ClosingState());
        context.getLiftState().close();
    }
    public open = () => console.log("電梯門開啟...");
    public run = () => {};
    public stop = () => {};
}
export class ClosingState extends LiftState {
    constructor() {
        super();
    }
    public close = () => console.log("電梯門關閉...");
    public open = () => {
        context.setLiftState(new OpenningState());
        context.getLiftState().open();
    }
    public run = () => {
        context.setLiftState(new RunningState());
        context.getLiftState().run();
    };
    public stop = () => {
        context.setLiftState(new StoppingState());
        context.getLiftState().stop();
    };
}
export class RunningState extends LiftState {
    constructor() {
        super();
    }
    public close = () => {};
    public open = () => {};
    public run = () => console.log("電梯上下運行...");
    public stop = () => {
        context.setLiftState(new StoppingState());
        context.getLiftState().stop();
    };
}
export class StoppingState extends LiftState {
    constructor() {
        super();
    }
    public close = () => {};
    public open = () => {
        context.setLiftState(new OpenningState());
        context.getLiftState().open();
    };
    public run = () => {
        context.setLiftState(new RunningState());
        context.getLiftState().run();
    }
    public stop = () => console.log("電梯停止了...");
}
(() =>{
    const context = new Context();
    context.setLiftState(new ClosingState());
    context.open();
    context.close();
    context.run();
    context.stop();
})()
電梯門開啟...
電梯門關閉...
電梯上下運行...
電梯停止了...
```

### 回顧
* Client場景變簡單了，只要給初始狀態，調用相關方法，完全不用考慮狀態的切換變更。也就是說只看到行為的發生改變，並不用知道是狀態變化引起的改變。
* 各場景的程式碼縮短了，因為切成各個子類別; 也取消了switch…case的判斷。
* 符合”開閉原則”, 因為要增加狀態，除了要增加子類別，也要修改原有的類別，只是要在原有的方法上增加新的方法，而不更動原有的。
* 符合單一職責, 現在各狀態式單獨的類別，只有與這狀態相關的因素能做修改。

## 狀態模式的定義
`Allow an object to alter its behavior when its internal state changes.
The object will appear to change its class.
(當一個對象內在狀態改變時，允許其改變行為，這個對象則看起來像是改變了類別)`
![](/images/DP/3ANdROM.jpg)

* State - 抽象狀態
    * interface或abstact class, 負責對象狀態定義，並且封裝環境腳色用來實現狀態切換。
* ConcreateState - 具體狀態
    * 自己的狀態行為管理，跟趨向狀態處理; 當下狀態要做的事情，以及過渡到其他狀態。
* Context - 環境腳色   
    * 定義client要得介面，並且負責具體狀態的切換。

#### Pros :
* 結構清晰 : 避免過多switch…case或if…else，增加可維護性
* 遵循設計原則 : 實現”開閉原則” 和 “單一職責”
* 封裝性良好 : 狀態變換放置到內部實現，外部不用知道  

#### Cons :
* 子類別太多 = 類別膨脹

#### 使用場景
* 行為隨著狀態改變而改變
* 條件、分之判斷語句的替代

#### 組合技
建造者+狀態模式 : 將狀態間切換的一定順序用建造者做構建。