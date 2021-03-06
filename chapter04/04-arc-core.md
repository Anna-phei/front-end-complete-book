## 第4章: 前端架构核心思想

掌握牢固的基础是了解架构（或js框架）关键。正所谓“工欲善其事，必先利其器”。如果你有读过js库或框架的经验，你是否有过因为架构基础不牢固而觉得代码晦涩难懂。很明显本章内容将是你需要的。
本章将围绕以下内容进行详细阐述：

1. 框架中常用的6种设计模式。
2. 作为一个标准的前端开发，需要了解的v8知识点。
3. 宏任务和微任务的处理过程是怎么样的。
4. 介绍有哪些异步加载规范。
5. 函数式编程基础
6. 以v-dom和状态原理两个例子入手，深入剖析两者的实现过程。

### 4.1 常用设计模式介绍

#### 4.1.1状态模式

状态模式是一个比较有用的模式，意思是指当一个对象的内部状态发生变化时，会产生不同的行为。

比如说某某牌电灯,按一下按钮打开弱光, 按两下按钮打开强光, 按三下按钮关闭灯光。在我们的想象中，基本的模型应该是如下描述

![](images/4-1-2.png)

状态模式允许对象在内部状态改变时，改变其行为，从对象的角度看好像进行了改变。实际开发中，某处文字可能和模型中的一字段进行关联，根据某一个状态显示不同的内容，这时候状态模式可能是你需要的（当然 switch-case, if-else可以继续）。

状态模式中有几个角色，分别是Context，State，各个子状态的实现。Context中保存了Client端的操作接口，同时也保存子状态的实现，代表着当前状态。抽象类State声明了子状态应该实现的各个方法。

![](images/state.jpg)

先看下Context的实现

```ts
export default class Context {
    private state: State;
    constructor(state: State) {
        this.transitionTo(state);
    }

    public transitionTo(_s: State): void {
        console.log(`Context: transition to ${(<any>_s).constructor.name}`);
    this.state = _s;
    this.state.setContext(this);
    }

    public setSlighLight(): void {
        this.state.slightLight();
    }

    public setHightLight(): void {
        this.state.highLight();
    }

    public close(): void {
        this.state.close();
    }

}
```

在transitionTo方法中改变当前状态，参数为实例化的子状态类。

再看下State的实现及其SlightLightClass的实现，为了篇幅考虑，我们在这里只贴出部分的代码，完整的代码参考[https://github.com/houyaowei/front-end-complete-book/tree/master/chapter04/code/4.1DesignPattern/State](https://github.com/houyaowei/front-end-complete-book/tree/master/chapter04/code/4.1DesignPattern/State)。

```ts
export default abstract class State {

    protected context: Context;
    public setContext(_c: Context) {
        this.context = _c;
    }

    public abstract slightLight(): void;
    public abstract highLight(): void;
    public abstract close(): void;    

}
```

> 请注意，如果对TypeScript的抽象类语法还不是很理解的话可以参考官网的class部分。

```ts
export default class SlighLightClass extends State {

    public slightLight(): void {

        console.log("state in SlighLightClass, I will change state to         highLight");
    //切换到新的状态
    this.context.transitionTo(new HighLight());
    }

    public highLight(): void {
        console.log("hightstate state in SlighLightClass");
    }

    public close(): void {
        console.log("close state in SlighLightClass");
    }

}
```

我们来测试下：

```ts
import Context from "./Context";
import SlightLight from "./SlightLightClass";
import CloseLight from "./CloseClass";

// const context = new Context(new SlightLight());
//我们先用close状态初始化
const context = new Context(new CloseLight());
context.close();
context.setSlighLight();
context.setHightLight();
```

结果如下：

`

Context: transition to ColseClass

state in closeClass, I will change state to slight

Context: transition to SlighLightClass

state in SlighLightClass, I will change state to highLight

Context: transition to HighLightClass

highLight state in HighLightClass

`

现在我们把初始状态调整为SlightState,重新编译、运行

`

Context: transition to SlighLightClass

state in SlighLightClass, I will change state to highLight

Context: transition to HighLightClass

highLight state in HighLightClass

state in hightLight, I will change state to close

Context: transition to ColseClass

`

状态模式封装了转换规则，并枚举了可能的状态。将所有的与某个状态有关的行为放到一个类中，所有可以方便地增加状态。

状态模式的使用必然会增加系统类和对象的个数。 状态模式的结构与实现都较为复杂，如果使用不当将导致程序结构和代码的混乱。 状态模式对"开闭原则"的支持并不太好，对于可以切换状态的状态模式，增加新的状态类需要修改那些负责状态转换的源代码，否则无法切换到新增状态，而且修改某个状态类的行为也需修改对应类的源码。



#### 4.1.2策略模式

策略模式是一种行为模式，它允许定义一系列算法，并将每种算法分别放入独立的类中，在运行时可以相互替换，主要解决的是在多种算法相似的情况下，尽量减少if-else的使用。

策略模式的应用比较广泛，比如年终奖的发放，很多公司都是根据员工的薪资基数和年底绩效考评来计算，比如说以3、6、1方式为例，绩效为3的员工年终奖有2倍工资，绩效为6的员工年终奖为1.2倍工资。

今天我们以一个字符串操作为例进行解释，对数组的sort和reverse方法，定义两种策略，根据需要的策略实例执行对应的策略。

先声明一个策略接口：

```ts
interface Strategy {
     toHandleStringArray(_d: string[]): string[];
}
```

再实现Sort方法的策略实现（这里我们不讨论sort方法的缺陷）

```ts
class StrategyAImpl implements Strategy {

    public toHandleStringArray(_d: string[]): string[] {
        //其他业务逻辑
        return _d.sort();
    }

}
```

接下来看看reverse方法的策略实现

```ts
class StrategyBImpl implements Strategy {

    public toHandleStringArray(_d: string[]): string[] {
        //其他业务逻辑
        return _d.reverse();
    }
}
```

你也许已经发现了，如果我们想实现数组的其他策略，只需要实现对应的接口即可。即能在不扩展类的情况下向用户提供能改变其行为的方法。

到现在还缺少一个关键的零件，改变策略的载体--Context类。

```ts
class Context {
    private strategy: Strategy;
    constructor(_s: Strategy) {
    this.strategy = _s;
}

public setStrategy(_s: Strategy) {
    this.strategy = _s;
}

//执行的方法是策略中定义的方法
public executeStrategy() {

    //标明是哪个策略类
    console.log(
`Context: current strategy is ${(<any>this.strategy).constructor.name}`
    );
     const result = this.strategy.toHandleStringArray(names);
     console.log("result:", result.join("->"));
    }

}
```

我们来测试一下，看效果是怎么样的，先假定Context类中的数组是如下形式的：

```ts
const names: string[] = ["hou", "cao", "ss"];
```

现在开始实例化reverse方法的策略

```ts
const context = new Context(new StrategyB());

context.executeStrategy();
```

效果如下：

`

Context: current strategy is StrategyBImpl

result: ss->cao->hou

`

再把策略方式切换到Sort，效果会变成这样的

`

Context: current strategy is StrategyAImpl

result: cao->hou->ss

`

这样的策略实现是不是挺方便的，”任你策略千千万，我仍待你如初恋“。策略模式的程序至少由两部分组层，第一部分是一组策略类，这里面封装了具体的算法，并负责算法的完整实现。第二部分是上下文(即Context)，上下文接受客户端的请求，随后被分发到某个策略类。

策略模式借助组合等思想，可以有效地避免许多不必要的复制、粘贴。同时，对开放-封闭原则进行了完美的支持，将算法封装到具体的策略实现中，易实现、易扩展。

状态可被视为策略的扩展，两者都基于组合机制。它们都通过将部分工作委派给“帮手”来改变其在不同情景下的行为。策略模式下这些对象相互之间完全独立。 状态模式没有限制具体状态之间的依赖，且允许它们自行改变在不同情景下的状态。

#### 4.1.3适配器模式

适配器模式的作用是解决两个接口不兼容的问题。将一个类的接口，转换成客户期望的另一种接口，能够达到相互通信的目的。

现实中，适配器的应用比较广泛。比如说港版的电器插头比大陆版的插头体积要大一些，如果你从香港买了一台Macbook Pro, 我们会发现电源插头是无法插到家里的插座上的，为了一个插头而改造家里已经装修好的插座显然不太合适，那么适配器就显得很有必要了。

还有就是如果你买了一台新版的Macbook Pro, 你发现外接设备的接口类型全部是Type-c,

如果想添加一个非Magic Mouse 鼠标，那么也不好意思，你也需要一个适配器。

下面我们通过简单的代码来描述下该模式。

我们想实现一个音乐播放的方法，常见的音乐文件格式有很多种，如Mp3,Mp4,Wma,Mpeg, RM, Vlc等。音乐播放文件也是五花八门，MediaPlay, 千千静听， RealPlay.....。播放器也不是万能的，不能播放所有的文件格式。

> 维基百科关于音频文件的描述[https://zh.wikipedia.org/wiki/%E9%9F%B3%E9%A2%91%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F](https://zh.wikipedia.org/wiki/%E9%9F%B3%E9%A2%91%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F)
> 
> 维基百科关于视频格式的描述
> 
> [https://zh.wikipedia.org/wiki/%E8%A7%86%E9%A2%91%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F](https://zh.wikipedia.org/wiki/%E8%A7%86%E9%A2%91%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F)

![](images/adapter.jpg)

我们先定义一个通用的播放接口

```ts
export default interface Target {
    play(type: string, fileName: string): void;
}
```

play方法需要两个参数，类型和文件名。因为我们要根据文件类型做适配，所有这个参数很有必要。

播放接口要支持最常见的音乐文件格式（如Mp3），当然也要支持更丰富的格式，至少可以看个视频吧，体验立马不一样了啊。

我们先定义一个高级播放接口

```ts
export default interface AdvanceTarget {

    playVlcType(fileName: string): void;

    playMp4Type(fileName: string): void;

}
```

实现两个具体的播放类，一个播放VLC格式的，一个播放Mp4格式的。

```ts
export default class VlcPlayer implements AdvancePlayer {

    public playVlcType(fileName : string) : void {

        console.log(`${fileName} is palying!`);

}

    public playMp4Type(fileName : string) : void {

        //假定Vlc播放器不能播放mp4格式

    }

}
```

```ts
export default class Mp4Player implements AdvancePlayer {

    public playVlcType(fileName: string): void {
        // 假定mp4播放器不支持VLC格式播放
    }

    public playMp4Type(fileName: string): void {
        console.log(`${fileName} is palying`);
    }

}
```

是时候实现适配器类的时候，以便更好解释适配器是如何架起两种接口的。

```ts
class MediaAdatper implements Target {

    private advanceTarget: AdvanceTarget;

    constructor(type: string) {
        if (type === "vlc") {
            this.advanceTarget = new VlcPlayer();
        }
        if (type == "mp4") {
            this.advanceTarget = new Mp4Player();
        }
    }

public play(type: string, fileName: string): void {

    if (type === "vlc") {
        this.advanceTarget.playVlcType(fileName);
    }
    if (type == "mp4") {
        this.advanceTarget.playMp4Type(fileName);
    }

    }

}
```

适配器类中持有高级接口的引用，根据文件类型初始化相应的类。所以在play方法就有了相应的实例，可以调用具体的方法。

现在，我们初始化好了适配器，主角播放器也该上场了，是到播放音乐的时候。

```ts
class Player implements Target {

    mediaAdapter : MediaAdapter;
    play(type : string, fileName : string) : void {
        if(type == "mp3") {
          //mp3直接播放
        } else if (type === "vlc" || type == "mp4") {
            this.mediaAdapter = new MediaAdapter(type);
            this.mediaAdapter.play(type, fileName);
        }

    }

}
```

下面我们进行下测试，

```ts
const player = new Player();

player.play("mp4", "笑看风云.mp4");
player.play("vlc", "烟雨唱扬州.vlc");
player.play("mp3", "背水姑娘.mp3");
player.play("wma", "左手指月.mp3");
```

测试结果：

`

笑看风云.mp4 is palying

烟雨唱扬州.vlc is palying!

Mp3 as the basic format, can play at will

sorry,type wma is not support

`

从上面的测试结果可以看出，两个不同的接口可以在一起愉快地通信了。爽歪歪。

最后我们总结下适配器的优点：

将接口或者数据转换代码分离了出来，代码看起来非常清晰。也同样遵循开闭原则，能在不修改现有客户端代码的情况下在程序中添加新类型的适配器。

适配器模式使整体复杂度增加，这是因为你每增加一种需要适配的类型，都要增加相应的接口和实现类。

#### 4.1.4观察者模式

观察者模式（Observer Pattern）又叫做发布-订阅模式(Pub/Sub)模式或消息机制。帮你的对象知悉现状，能及时响应订阅的事件，可以看成是一种一对多的关系。当一个对象的状态发生改变时，所有依赖它的对象都应得到通知。
观察者模式是松耦合设计的关键。
我们用淘宝购物中的一个例子来理解观察者模式。
你在淘宝上找到一款心仪的电脑，是最新发布的16寸的Mackbook Pro,但是联系卖家后发现没货，鉴于商铺比较好的信誉度和比较大的优惠力度，你觉得还是在这家买比较划算，所以就问卖家什么时候有货，商家告诉你需要等一周左右，还友情提示你:"亲，你可以先收藏我们的店铺，等有货了会再通知你的"，你收藏了店铺。电脑发烧友可不止你一个，小明、小华等陆陆续续也都收藏了该店铺。</br>
从上面的故事中可以看出，这是一个典型的观察者模式，店铺老板是发布者，你、小明、小华都是订阅者。Mac电脑到货(即状态改变)，会依次通知你、小明，小华等，使用旺旺等工具依次给他们发布消息。

下面我们看下基本的模型：

![](images/4-1-1.png)

在上面的模型中可以看出，商家维护着和各位客户的引用关系，通过观察者添加、解除引用关系，就好比说，某天某客户不再中意这款电脑，商家就再无引用这份关系了。

> 本书中所有的代码均是由Typescript描述，众所周知，Typescript为Js的超集，具有强类型约束，在编译期就可以消除安全隐患，具体的介绍可以参考管网，[https://www.typescriptlang.org/](https://www.typescriptlang.org/), 也可以联系笔者可以共享的电子书

下面我们看下代码模型，先看下商家的代码实现：

```ts
import Customer from "{path}/CustomerModal";

export default class Seller {

    customers: Customer[];
    register(customer): void {
        this.customers.push(customer);
    }

    remove(id: number): void {
        this.customers.forEach(c => {
        if (c.getId() === id) {
            console.log(`this id: ${id} should be removed`);
        }
        });
    }

    notifyAll(): void {
        this.customers.forEach(cus => {
        cus.dealOrder();
    });
    }

}
```

customers属性维护着所有订阅者，数组中的 每个元素都是Customer对象，我们从模拟对象出发，抽象出该对象：

```ts
export default class Customer {

    private id: number;
    private name: string;
    private address: string;
    private telNum: string;
    private orders: Order[];

    constructor(_id: number, _name: string, _address: string, _telNum: string) {

        this.id = _id;
        this.name = _name;
        this.address = _address;
        this.telNum = _telNum;
    }

    getId(): number {
        return this.id;
    }

    dealOrder(): void {
        //make a order
        console.log(`I am + ${this.name}， I have got message from seller`);
    }

}
```

看了商家的模型后，来看下观察者模式的模型：

```ts
import Seller from "./Seller";

import Customer from "./CustomerModal";

export default class Observer {

    constructor() {
        this.seller = new Seller();
    }
    private seller: Seller;

    register(customer: Customer): void {
        console.log("");
        this.seller.register(customer);
    }

    fire(): void {
        this.seller.notifyAll();
    }

    remove(customerId: number): void {
        this.seller.remove(customerId);
    }

}
```

上面的代码中，是从OOP的实现方式出发进行设计。已经有了观察者模式所需要的两个主要元素：主题（商家）和观察者（各位客户），一旦数据改变，新的数据就会以某种形式推送到观察者的手上。

现在我们来测试下这几段代码：

```ts
let customer1 = new Customer(1101, "caozn", "shanxi", "12900000");
let os = new Observer();
os.register(customer1);

let customer2 = new Customer(1102, "houyw", "henan", "12900001");
os.register(customer2);
console.log(os.getAllCustomers().length);

os.fire();
```

得到的结果如下：

`

现在商家有 2 个客户订阅

I am caozn， I have got message from seller

I am houyw， I have got message from seller

`

主题和观察者之间定义了一对多的关系。观察者依赖整个主题（商家），毕竟要从主题那里获得通知。并且主题是具有状态的，也可以控制这些状态。

观察者模式定义了主题和观察者之间的松耦合关系，并且还可以让两者进行交互，而不用太关注对方的细节。"keep it simple".当然缺点也不是完全没有的， 如果过多的使用发布订阅模式, 会增加维护的难度。

#### 4.1.5代理模式

代理模式是一种结构性模式，作用是提供一个中间对象，为其他对象提供控制这个对象的能力。

代理模式在现实的生活中有很多的实例，信用卡是银行账号代理，银行账号则是一捆一捆现金的代理，他们都有相同的功能(接口)-付款。信用卡的付款方式让用户和商户都比较满意，用户不用随身携带大量的现金，商户也因为交易收入能以电子化的方式进入银行账户中，  无需担心存款时出现现金丢失或被抢劫的情况，减少很多的麻烦。

影视剧86版《西游记》中也有代码模式的影子，在第七集《计收猪八戒》中，孙悟空为了给高家“除妖”，扮成高翠兰的模样。从代理模式的角度看，对高翠兰的外貌和行为抽象成接口，高小姐和孙悟空都实现这个接口，孙悟空就是高小姐的代理类。

我们的例子以一个求婚为原型进行说明，一位男士想向他女朋友求婚，但由于各种原因不好意思说出口，就想请他的好朋友（办大事得找个靠谱的朋友）帮忙转达意思。

![](images/proxy.jpg)

先看下接口

```ts
interface Subject {
    proposal() : void;
}
```

实现类

```ts
class RealSubject implements Subject {
    public proposal(): void {
        console.log("Darling, Can you marray me?");
    }

}
```

代理类

```ts
class Proxy implements Subject {

    private realSubject : RealSubject;

    private chcekIsGoodFriend() : boolean {
        console.log("It's is checking if good friend");
        const r = Math.ceil(Math.random() * 10);
        //只有够意思才给你传话
        if (r > 6 || r == 6) {
            return true;
        } else {
            return false;
        }

    }

    private checkPromission() {
        console.log("It's checking the promission");
        if (this.chcekIsGoodFriend()) {
            return true;
        }
        return false;
    }

    public proposal() : void {
         if(this.checkPromission()) {
            this.realSubject.proposal();
        }

    }

}
```

帮忙传话需要征得当事人的同意(checkPromission)，还是个靠谱的朋友(chcekIsGoodFriend)。这两个条件具备了，这事儿就会靠谱很多。

现在我们测试下：

```ts
let realSubject = new RealSubject();

let subject : Subject = new Proxy(realSubject);

subject.proposal();
```

请自行检测啊，看某位朋友靠不靠谱，同意还是不同意。

代理模式的优缺点：

代理模式可以代理目标对象，并且是在毫无绝唱的情况下进行。可以在不对服务或客户端做出修改的情况下创建新代理

#### 4.1.6装饰者模式

装饰者模式能够在不改变对象自身的前提下，在运行期给对象增加额外的功能。提到增强对象，我们的第一印象可以使用继承扩展这个类，但是不能否认的是继承还是有如下几个问题：

1、继承是静态的，无法在运行时更改已有对象的行为。

2、在某些编译型语言中(如Java)是不允许同时继承多个类的。

下面我们通过一个简单的示例进行说明：

![](images/decorator.jpg)

在接口中，定义一个基本draw方法

```typescript
interface Shape {

    draw(): void;

}
```

然后实现两个类实现该接口：

```ts
CircleShape implements Shape {

    public draw(): void {
       console.log("the drow method in class CircleShape");
    }
}

class RectangleShape implements Shape {

    public draw(): void {
        console.log("the drow method in class RectangleShape");
    }

}
```

接下来定义装饰类的基类

```ts
class ShapeDecorator implements Shape {

    protected shape: Shape;

    constructor(s: Shape) {
        this.shape = s;
    }

    public draw() {
        this.shape.draw();
    }

}
```

protected属性保存着Shape对象的引用，调用draw方法，就调用该对象的该方法。接下来定义扩展后的装饰类。

```ts
class BlueShapeDecorator extends ShapeDecorator {

    public draw(): void {
        super.draw();
        this.setBGImage();
    }

    private setBGImage(): void {
        console.log("set background Image im BlueShapeDecorator");
    }

}

class GreenShapeDecorator extends ShapeDecorator {

    public draw(): void {
        super.draw();
        this.setBorder();
    }

    private setBorder(): void {
        console.log("set border in GreenShapeDecorator");
    }

}
```

万事俱备只欠东风，下面进行下测试：

`

let shape:  Shape  =  new  CircleShape();

let shape2:  Shape  =  new  RectangleShape();

const decorator1 =  new  BlueShapeDecorator(shape);

const decorator2 =  new  GreenShapeDecorator(shape2);

decorator1.draw();

console.log("---------------------");

decorator2.draw();

`

输出结果：

`

the drow method in class CircleShape

set background Image im BlueShapeDecorator

---------------------

the drow method in class RectangleShape

set border in GreenShapeDecorator

`

按照老规矩还是要总结下装饰者模式：装饰者和被装饰者相对对立，不是相互耦合。装饰者模式是继承的替代品，可以动态扩展出一个实现类。

### 4.2 V8引擎该了解的

V8,这个词对很多前端码农来说，即熟悉又陌生。熟悉的是它师出名门，是由Google开发的高性能引擎。我们熟悉nodejs是基于这个开发的，对前端开发比较友好的Chrome浏览器也是有它的影子，甚至它也实现了前端的ECMAScript和WebAssembly规范。陌生的是它就想个黑盒子。在这一小节中我们了解下它的基本情况（这块的表达再需要调整下）。

V8可以运行在各主流平台，windows， MacOS，Linux，Android ，IOS。它是可以单独运行的，并且能嵌入到其他的C++程序中。

V8是Webkit的子集。关于这个由来，我们需要简单介绍下二次浏览器大战。

1993年，浏览器Mosaic诞生，它是由Marc Andreessen领导的团队开发，这就是后来鼎鼎大名的Netscape（网景）浏览器。该浏览器只能显示静态的HTML元素，不支持js和CSS。但是并不妨碍它当时收到网民的欢迎，在世界范围内得到认可。市场占有率甚至达到90%。然而，事情从1995年开始变的有些不一样，这一年微软推出了IE，鉴于和windows绑定的天然优势，IE获得空前的成功，并逐渐取代了网景浏览器。Netscape的市占率从1990年代中期的90%也下降至2006年底的不到1%，并于2008年初停止对网景浏览器的研发。网景公司也从1998年开始成立Mozilla基金会，重新发力，研发FireFox（火狐浏览器），并于2004发布1.0版，正式拉开第二次浏览器大战的序幕。IE也因为自身发展较慢，Firefox自推出以来因为插件丰富、功能完善、兼容性更好，市场占有率不断攀升。

浏览器之间的纷纷扰扰好像永无停歇。2003年，苹果公司发布了Safari浏览器，并在2005年开源了该浏览器的内核，发起了一个叫做Webkit的开源项目。浏览器的春天来了。2008年，Google以Webkit为内核，创建了一个新的项目叫Chromium（管网：http://www.chromium.org/  这个本身也是一个浏览器）,在这个项目的基础上发布了自己的浏览器产品Chrome，Chromium像一个开源实验室，它会尝试较新的技术，等这些技术稳定了，Chrome才会把他们集成进来。

2013年，Google宣布了Blink内核，这个其实是从Webkit复制出去独立运作的。其中的原因是Google和苹果公司之间有了一些分歧。Webkit将于Chromium相关的代码删除了，同时，Blink将除了chromium需要的之外的移植代码也进行了删除。Chrome从28.0.1469.0正式版本开始正式使用该引擎。

#### 4.2.1 webkit架构

我们先看下Webkit的大致架构图：

![](images/webkit-arch.jpg)

webkit一个比较显著的特征就是它支持不同的浏览器，甚至微软的Edge也加入到了这个阵营。虚线部分表示该模块在不同浏览器中使用的webkit内核实现可能是不一样的。实线部分标记的模块表示它们基本上是共享的。

webkit中默认js引擎指的是JS Core,而在Chromium中则是如雷贯耳的V8。什么是JavaScript引擎，就是能够将JavaScript代码处理并执行的运行环境。渲染引擎提供了渲染网页的功能，渲染引擎主要包含HTML解析器、css解释器、布局和JavaScript引擎。

![](images/render.png)

- HTML解析器：主要负责解释HTML元素，将HTML元素解析成DOM。

- CSS解释器：负责解析css文本，计算DOm中各元素样式信息，为布局提供样式基础。

- 布局：把DOM信息和css信息结合起来，计算出它们的元素大小及位置信息，形成具有所有信息的表示模型。

- JavaScript引擎：使用JavaScript可以操作网页的内容，也能修改css的信息，JavaScript引擎能够解释JavaScript代码，可以操作网页内容及样式信息。

上面的表述还是比较抽象，我们是用一张图来描述下网页是怎么呈现到客户眼前的

![](images/webkit-render.png)

> 虚线表示渲染过程中，该阶段可能依赖其他模块。比如在网页的下载过程中，需要使用到网络和存储。

##### 4.2.2 隐藏类和对象表示

1、隐藏类

虽然在JavaScript没有类型的明确定义，但是V8是在C++的基础上开发完成的，也是可以为JavaScript的对象构造类型信息的。V8是借用了类和偏移位置的思想，将本来通过属性名匹配来访问属性值的方法进行了改进，使用类似C++编译器的偏移位置机制来实现，这就是隐藏类。

隐藏类将对象分为不同的组，在同一个组的对象如果有相同的属性名和属性值的情况下，会将这些属性名和对应的偏移位置保存到一个隐藏类中。这样介绍还是比较抽象，下面通过一个简单的例子解释：

![](images/obj.png)

创建了两个对象p1,p2, 他们有相同的属性name和age.在v8中，它们被“安排”到同一个组（即隐藏类中），并且这些属性有相同的偏移值。这样p1和p2可以共享这个类型信息。访问这些属性时就根据隐藏类的偏移值就可以知道它们的值继而访问。如果你再给某一个对象运行时添加属性时，比如加入一下代码：

`p1.address='xi，an'`

那么，p1对应的就是一个新的隐藏类。

我们了解了隐藏类，下面看下代码是如何使用这些隐藏类来高效访问对象的属性的。我们以以下代码进行说明：

```js
function getName(person){
 if(person && person.name){
 return person.name
 }
}
```

访问的基本过程是这样的：首先获取隐藏类的地址，然后根据属性值查找偏移值，计算出属性的地址。不过遗憾的是，这个过程是比较耗时的。那么是否可以使用缓存机制呢？答案是肯定的，这套缓存机制叫做内联缓存(inline-cache)，主要思想就是将使用之前查找的结果缓存起来，避免方法和属性被存取时出现的因哈希表查找带来的问题。

上面的getName方法中，为了查找name属性，如果未缓存，则退回到前面说的通过查找哈希表的方法查找。否则直接读取缓存。

##### 4.2.3、对象在内存中的表示

JavaScript中有6种基础类型，分别是String, Number, Boolean, Null, Undefined, Symbol,这些类型的值都有固定的存储大小，往往都会保存到栈中，由系统自动分配存储空间。我们可以直接按值访问这部分的值。其他类型为引用类型，比如对象，内存中分配值就不是固定的。该类型的变量值是保存到堆（堆是非结构化区域，堆中的对象占用分配的内存。这种分配是动态的，因为对象的大小/寿命/数量是未知的，所以需要在运行时分配和释放）内存中的，这部分的值是不允许我们直接访问的。

```js
var name="houyw";
var age = 23;
var isMale = true;
var empty = null;

var person = {
 name: "houyw",
 age: 23,
 isMale: true;
}
```

![](images/stackAndHeap.png)

上面是变量在内存分配空间的宏观观察，具体的实现细节对我们来说还是个黑盒。不急，接下来我们就会详细介绍下具体的过程。

默认情况下JavaScript对象会在堆上分配固定大小的空间存储内部属性，预分配空间不足时(无空闲slot)，新增属性就会存储到properties中。而数字存储在element中。如果properties和elements空间不足时，会创建一个更大的FixedArray。为了便于说明问题，我们举例说明：

```js
var obj ={};
```

![](images/jsobject-1.png)

> V8使用map结构来描述对象。map可以理解为像table一样的描述结构。

通过对象字面量创建的空属性对象默认分配4个内部属性存储空间。

继续添加两个属性：

```js
obj.name="houyw";

obj.age =23;
```

![](images/jsobject-2.png)

name和age属性默认存储到对象的内部属性中。再添加两个数字属性：

```js
obj[0]="aaa";

obj[1] = "bbb";
```

![](./images/jsobject-3.png)

##### 4.2.4 内存管理

v8的内存管理只要有两点：一是V8内存的划分，二是V8对于JavaScript代码的垃圾回收。

V8的内存划分如下：

- Zone类：该类主要管理小块内存。当一块小内存被分配之后，不能被Zone回收，只能一次性回收Zone分配的所有小块内存。当一个过程需要很多内存，Zone将需要分配大量的内存，却又不能及时释放，结果导致内存不足。

- 堆：管理JavaScript使用的数据、生成的代码、哈希表等。为方便实现垃圾回收，堆被分为三个部分：
  
  ![](./images/js-gc.png)
  
  需要说明的是年轻分代，主要是为新创建的对象分配的内存，新创建的对象很容易被回收，为了方便垃圾回收，使用了复制的形式，将年轻分代分成两半，一般用来分配，另一半在回收的时候负责将之前负责保留的对象复制过来。对年老分代，主要是将年老分代的对象、指针等数据在使用内存较少时进行回收。而对于大对象空间来说，主要是用来为哪些需要较多内存的大对象分配的，需要注意的是每个页面只分配一个对象。

V8使用了精简整理的算法，用来标记那些还有引用关系的对象，然后回收没有被标记的对象。最后对这些对象进行整理和压缩。

那么垃圾回收机制是怎么判断对象是否存活呢？如何检查对象的生死，是通过客户机或者程序代码是否可以触达此对象。可触达性(Reachability)还可以这么理解：另一个对象是否可以获得它，如果可以的话，该对象所需的内存会被保留。

#### 

#### 4.3 宏任务和微任务

我们都知道JavaScript是单线程语言，单线程意味着，JavaScript代码执行的时候，都只能一个主线程来处理所有的任务。

单线程是有必要的，最主要的原因是最初也是最主要的执行环境---浏览器。我们面对的是各种各样的操作DOM、操作CSS。如果是多线程执行环境，很难保证在频繁操作的情况下保证一致性，退一步讲，即使保证了一致性，那么对性能也会有比较大的影响。

后来，为了实现多线程，引入了webworker,但是该技术的使用却有诸多限制：

1、新线程都受主线程的完全控制，不能独立执行，只能是隶属于主线程的子线程，

2、子线程并没有操作I/O的权限

尽管js是单线程的，单也是“非阻塞”的，js是怎么实现的呢？其实就是由和本小结相关联的event loop(在这里我们主要讨论浏览器版的实现)实现。

脚本执行的时候 ，js引擎会解析代码，并将其中同步执行的代码依次加入到执行栈中，从头开始执行。如果当前执行的是一个方法，那么js会向执行栈中添加这个方法的执行环境，然后进入这个执行环境继续执行其中的代码。当这个执行环境中的代码 执行完毕并返回结果后，js会退出这个执行环境并把这个执行环境销毁，回到上一个方法的执行环境。这个过程反复进行，直到执行栈中的代码全部执行完毕。

![](images/stack.png)

上面也提到了，执行栈中存放的是同步代码。那么当异步代码执行时情况又是怎样的呢？既然是非阻塞式的，那么又是通过什么机制保证的呢？这里不得不提到事件队列(Task Queue)。

熟悉JavaScript的都知道，对于异步请求，js引擎可能并不会及时返还结果，而是将这个事件挂起，继续执行执行栈的其他方法。当异步请求返还结果时，js引擎会将这个事件推入到另一个队列，叫做事件队列。放入该队列的时间并不会立即执行，而是要等到执行栈中无可执行，主线程处于空闲状态时，主线程会去查找事件队列是否有任务。如果有，主线程会取出第一位的事件，并把事件对应的回调函数放入到执行栈中，然后执行。如此反复，就形成了

无限执行的过程。这个过程就被称为"事件循环(Event Loop)"。

有了上面的介绍，就更有利于我们去介绍宏任务和微任务。

上面我们介绍过对异步请求，js引擎才会把这个事件推入到事件队列。看似和宏任务、微任务没有什么关系。实际上，js引擎会根据异步事件的类型，把对应的异步事件推到对应的宏任务队列或者微任务队列中。当主线程空闲时，主线程先会查看微任务队列中是否有事件，如果有就取出一条运行，如果没有，就从宏任务队列中取出一条执行。依次反复。

现在，我们总结一下：**同一次事件循环中，微任务的执行优先级会高于宏任务**。

下面我们通过代码实例来看下代码的执行情况，总结一下代码在哪种情况下是会被推入微任务队列，哪种情况下会被推入宏任务队列。

 我们先看个例子：

```js
console.log("console-1");

setTimeout(() => {
  console.log("settimeout-1");
  Promise
    .resolve()
    .then(() => {
      console.log("promise-1")
    });
});

new Promise((resolve, reject) => {
  console.log("promise-2")
  resolve("promise-2-resolve")
}).then((data) => {
  console.log(data);
})

setTimeout(() => {
  console.log("settimeout-2");
})

console.log("console-2");
```

我们先在浏览器中运行下看下执行结果：

> console-1
> promise-2
> console-2
> promise-2-resolve
> settimeout-1
> promise-1
> settimeout-2

我们看下执行过程，首先执行全局的代码:

```js
console.log("console-1");
```

![](images/task-1.png)

控制台打印：

 console-1

第二步执行，

```js
setTimeout(() => {
 //我们把这里命名为callback1
 console.log("settimeout-1");
 Promise
 .resolve()
 .then(() => {
 console.log("promise-1")
 });
});
```

![](images/task-2.png)

因为setTimeout会被认为是宏任务，所以会被加入到宏任务队列。打印结果仍然是：

> console-1

第3步：

```js
new Promise((resolve, reject) => {
 console.log("promise-2")
 resolve("promise-2-resolve")
}).then((data) => {
//这里定义为callback2
 console.log(data);
})
```

![](images/task-3.png)

Promise的构造函数是同步执行的，会立即执行。而then中的回调函数被认为是微任务，所以会被加入到微任务队列中。打印出结果：

> console-1
> promise-2

第4步：

```js
setTimeout(() => {
   //这里定义为callback3
  console.log("settimeout-2");
});
```

setTimeout中回调函数继续被推到宏任务。

![](images/task-4.png)

打印结果：

> console-1
> promise-2

第5步：

```js
console.log("console-2");
```

![](images/task-5.png)

打印结果：

> console-1
> promise-2
> 
> console-2

第6步：全局代码已执行完毕，现在到了js引擎从微任务队列中取出一个微任务执行。

```js
console.log(data);
```

执行then中的回调函数，数据为resolve后的值：

![](images/task-6.png)

打印结果：

> console-1
> 
>  promise-2
> console-2
> promise-2-resolve

第7步：微任务队列中只有一个任务，执行完后，后从宏任务队列中取出一个任务(callback1)执行。

```js
console.log("settimeout-1");
Promise
 .resolve()
 .then(() => {
 //这里定义为callback4
 console.log("promise-1")
 });
```

![](images/task-7.png)

打印结果：

> console-1
> 
> promise-2
> 
> console-2
> 
> promise-2-resolve
> 
> settimeout-1

执行完同步任务后，又遇到另一个Promise，异步执行完又在微任务队列中加入一条任务。

第8步：微任务队列中有任务，继续从该队列中取

```js
console.log("promise-1")
```

![](images/task-8.png)

执行结果：

> console-1
> 
> promise-2
> 
> console-2
> 
> promise-2-resolve
> 
> settimeout-1
> 
> promise-1

第9步：从宏任务取一个任务执行

```js
console.log("settimeout-2");
```

![](images/task-9.png)

执行结果：

> console-1
> promise-2
> console-2
> promise-2-resolve
> settimeout-1
> promise-1
> settimeout-2

我们再看一个例子是关于async/await的，该特性已经被广泛应用，所以有必要看下它的执行情况。

```js
setTimeout(() => console.log("setTimeout"))

async function main() {
  console.log("before resovle")
  await Promise.resolve();
  console.log("after resovle")
}
main()
console.log("global console");
```

async函数体内，在await之前的代码都是同步执行的，可以理解为给Promise构造函数内传入的代码，await之后的所有代码都是在Promise.then中的回调，会被推入到微任务队列中。

> before resovle
> 
> global console
> 
> after resovle
> 
> setTimeout

最后我们整理下浏览器端哪些方法的回调会被推入到宏任务和微任务队列中：

宏任务：I/O， setTimeout,，setInterval，requestAnimationFrame

微任务：Promise.then， catch， finally，await(后)

该部分的内容相对比较抽象，希望大家结合更多的资料去理解。这部分完全理解后，详细你对JavaScript的执行过程有更深的了解。

#### 4.4 异步加载规范

前端的模块化的热度一致没有退，因为模块化可以有更好代码约定，方便依赖关系的管理，规范前端开发规范，提高代码的可复用性。更重要的是模块化是前端工程化的重要因素，工程化是一种较高层次思维，而不是具体的某一项技术。所谓的前端工程化是把前端项目作为一个完整的工程进行分析、组织、构建、优化，在相应的过程中进行优化配置，达到项目结构清晰、边界清晰，各开发人员分工明确、配合默契、提高开发效率的目的。

模块化的贯彻执行离不开相应的约定，即规范。这个是能够进行模块化工作的重中之重，孟子说的“不以规矩不能成为方圆”也是这个道理。

下面，我们看下目前流行的前端模块规范：Amd，Cmd,ES6 module和CommonJS.

##### 4.4.1 Amd和requirejs

amd(Asynchronous Module Definition，规范地址：[https://github.com/amdjs/amdjs-api/wiki/AMD](https://github.com/amdjs/amdjs-api/wiki/AMD))，中文翻译为异步模块定义。该规范通过define方法定义模块，通过require方法加载模块。

RequireJS 想成为浏览器端的模块加载器，同时也想成为 Rhino / Node 等环境的模块加载器。。

我们先看下define的api定义:

```js
define(id?, dependencies?, factory);
```

该方法接受3个参数，？表示可选项，定义模块时可以不用指定。第一个参数表示该模块的ID，第二个参数dependencies表示该模块所依赖的模块，该参数是一个数组，表示可以依赖多个。第三个factory是一个函数（也可以是对象），既然是函数，就可以有参数，那么参数是怎么传递进来的呢？这时候我们再想起来dependencies这个参数了，依赖的声明顺序和该工厂函数参数的声明顺序保持一致，如下面的实例代码：

```js
define([ 'service', 'jquery' ], function (service, $) {
    //业务
}
   return { }

})
```

依赖模块service和jquery在工厂方法执行前完成依赖注入，即依赖前置。

下面我们看下完整的例子，我们以AMD规范的requirejs实现为例，我们先在HTML文件中加入：

```js
<script data-main="js/main" src="js/libs/require.js"></script>
```

data-main属性指定工程文件入口，在main.js中配置基础路径和进行模块声明：

```js
requirejs.config({
    //基础路径
    baseUrl: "js/",
    //模块定义与模块路径映射
    paths: {
        "message": "modules/message",
        "service": "modules/service",
        "jquery": "libs/jquery-3.4.1"

    }

})

//引入模块

requirejs(['message'], function (msg) {
    msg.showMsg()

})
```

再message模块中引入依赖模块service和jquery，

```js
define(['service', 'jquery'], function (service, $) {

    var name = 'front-end-complete-book';

    function showMsg() {
        $('body').css('background', 'gray');
        console.log(service.formatMsg() + ', from:' + name);
    }
    return {showMsg}

})
```

service代码如下：

```js
define(function () {

    var msg = 'this is service module';
    function formatMsg() {
        return msg.toUpperCase()
    };
    return {formatMsg}

})
```

详细的代码请参考代码实例。

##### 4.4.2 cmd和seajs

requirejs在声明依赖的模块时会在第一之间加载并执行。cmd（ Common Module Definition，通用模块定义,规范地址：[https://github.com/seajs/seajs/issues/242](https://github.com/seajs/seajs/issues/242)）是另一种模块加载方案，和amd稍有不同，不同点主要体现在：amd推崇依赖前置，提前执行。cmd是就近依赖，延迟执行。

Seajs 则专注于 Web 浏览器，同时通过 Node 扩展的方式可以在 Node 环境中运行。

> 扩展阅读：
> 
> seajs与requirejs的不同[https://github.com/seajs/seajs/issues/277](https://github.com/seajs/seajs/issues/277)，
> 
> seajs和requirejs对比[https://www.douban.com/note/283566440/](https://www.douban.com/note/283566440/)，
> 
> seajs规范文档[https://github.com/seajs/seajs/issues/242](https://github.com/seajs/seajs/issues/242)
> 
> require书写规范[https://github.com/seajs/seajs/issues/259](https://github.com/seajs/seajs/issues/259)

seajs官方：[https://github.com/seajs/seajs](https://github.com/seajs/seajs)，是cmd规范实现。规范部分不做详细的介绍，我们通过一个例子来说明：

和一般引入js文件的方法导入seajs支持,

```js
<script src="./js/libs/sea.js"></script>
```

> alias:当模块标识很长时，可以用这个简化，让文件的真实路径与调用标识分开。
> 
> paths: 当目录比较深，或需要跨目录调用模块时，可以使用 `paths` 来简化

下面对seajs做基本都配置，并声明模块

```js
seajs.config({

    charset: "utf-8",
    base: "./js/",
    alias: {
        jquery: "libs/jquery-3.4.1",
        message: "modules/message",
        service: "modules/service"
    },
    paths: {}
});

seajs.use("./js/main.js");
```

使用seajs.use方法在页面中加载任意模块， base指定seajs的基础路径，该属性结合alias中模块路径配置一起指向某一模块，这里需要注意的是路径的解析方法:

(1)相对标识

 在http://example.com/modules/a.js 中 require('./b')引入b模块，那么解析后的路径为: http://example.com/modules/b.js

(2)顶级标识

顶级标识**不以点（"."）或斜线（"/"）开始**， 会相对 SeaJS 的 base 路径来解析。假如base路径http://example.com/modules/libs/ ，在某模块中require('jquery/query-3.4.1')引入jquery模块，解析后的路径为 http://example.com/modules/libs/jquery/jquery-3.4.1.js

(3)普通路径

在某个模块中引入模块 require('http://example.com/modules/a') , 普通路径的解析规则，会相对当前页面来解析。

继续回到js/main.js中，引入message模块：

```js
define(function (require, exports, module) {

    require("message").showMsg();

})
```

message模块中serivce模块和jquery模块，

```js
define(function (require, exports, module) {

    var service = require("service");
    var $ = require("jquery");
    var name = 'front-end-complete-book';

    function showMsg() {
        $('body').css('background', 'gray');
        console.log(service.formatMsg() + ', from:' + name);
    }
    exports.showMsg = showMsg;
})
```

> 在seajs引入jquery模块需要做简单点改造，因为jquery遵循amd规范，所以需要做简单的改造，改造方式如下：
> 
> define(function(){
> 
>     //jquery源代码
> 
>     return jQuery.noConflict();
> 
> });

完整的代码请参考源码部分。

##### 4.4.3 Umd

兼容AMD和commonJS规范的同时，还兼容全局引用的方式, 规范地址：[https://github.com/umdjs/umd](https://github.com/umdjs/umd)  ， 常用写法如下：

`

```js
(function (root, factory) {    
 if (typeof define === 'function' && define.amd) {        
     //AMD        
     define(['jquery'], factory);
 } else if (typeof exports === 'object') {       
  //Node, CommonJS支持       
  module.exports = factory(require('jquery'));
 } else {        
    //浏览器全局变量(root 即 window)        
    root.returnExports = factory(root.jQuery);
 }}(window, function ($) {       
    function myFunc(){};
    //暴露公共方法    
    return myFunc;
}));
```

`

##### 4.4.4 ES6 module

ES6在语言标准的层面上引入了module，应该也更加规范。Es6 module编译时加载需要的模块，使用export或者export.default暴露出方法、类、变量，使用import导入需要的模块。

下面我们看个例子：

我们定义3个模块，moduleA, moduleB和moduleC, 其中moduleA作为主模块，在浏览器以module的方式导入。

```js
 import name, { msg, person } from "./moduleA.js";
```

在moduleA中可以导出各种数据结构：

```js
export var msg = "msg from moduleA";

var obj = {
  name: "hyw",
  age: 23
};

export { obj as person };

export default name = "module-A";
```

 可以把这些数据输出到页面上看看是否能被正确导入，

```js
<script type="module">
      import name, { msg, person } from "./moduleA.js";

      document.getElementById("test").innerHTML =
        msg + ",person name:" + person.name + ", module name is:" + name;
</script>
```

![](images/ES6-1.png)

可以在浏览器（Chrome,Safari,Opera, Firefox）正常执行。

另外，import方法返回Promise对象，所以也可以写成这样的：

```js
if (true) {
  import("./moduleB.js").then(res => {
    console.log(res.obj.name + ", module name:" + res.default);
  });
}

Promise.all([import("./moduleB.js"), import("./moduleC.js")]).then(
  ([moduleB, moduleC]) => {
    console.log( moduleB.obj.name + ", module name:" +
        moduleB.default + ", another module is :" + moduleC.default
    );
  }
);
```

moduleC模块中的代码较简单，

```js
export default name = "module-C";
```

刷新浏览器，可以输出：

`

czn, module name:module-B
 czn, module name:module-B, another module is :module-C

`

> 需要注意的是，ES6module输出的模块是引用，原始值发生变化，import加载的值也会跟着变。

##### 4.4.5 Commonjs

CommonJS(官网：[http://www.commonjs.org/](http://www.commonjs.org/)) 是以在浏览器环境之外构建 JavaScript 生态系统为目标而产生的项目，比如在服务器和桌面环境(nw.js, electron)中。前身叫做Serverjs，是由Mozilla的工程师Kevin Dangoor 在2009年1月创建的，在2009年正式更名为commonjs。

CommonJS 规范是为了解决 JavaScript 的作用域问题而产生，可以使每个模块在它自身的命名空间中执行。该规范的主要内容是，模块必须通过 module.exports 导出对外的变量或接口,通过 require() 来运行时加载其他模块的输出到当前模块中。

> 扩展阅读：Google group [https://groups.google.com/forum/#!forum/commonjs](https://groups.google.com/forum/#!forum/commonjs)

下面看一个简单点例子：

```js
module.exports = function(num) {
  if (typeof num != "number") {
    return 0;
  } else {
    return num * num;
  }
};
```

该模块中暴露一个方法，计算一个数字的平方，在主文件引入这个模块

```js
var square = require("./moduleA");
console.log(square(4));
```

> 补充一个知识点，exports和module.exports的关系：
> 
> 1. module.exports 初始值为一个空对象 ；
> 2. exports 是指向的 module.exports 的引用；
> 3. require() 返回的是 module.exports；

> 另一个需要注意点，commonjs模块输出到是值得拷贝，模块内部的变化不会影响到已经导出的值。

#### 

#### 4.5 函数式编程入门

函数式编程是一种编程范式，也就是说提供一种如何编写程序的方法论，主要思想是把运算过程尽量写成一系列嵌套的函数调用。常见的编程范式有命令式编程、函数式编程、面向对象编程、指令式编程等不同点编程范型。

在JavaScript中，函数作为一等公民，可以在任何地方被定义，无论是函数内还是函数外，还可以被赋值给一个变量，可以作为参数传递，也可以作为一个返回值返回。

##### 4.5.1引子

我们先通过两个简单的例子引入函数式编程，先看第一个：

想给数组中的每个元素进行平方计算

```js
let array = [1,2,3,4,5,6]

for(let i =0, iLen = array.length; i <iLen; i++){
  array[i] = Math.pow(array[i],2)
}
```

这是一个典型的命令式编程等例子，要一步一步说明该怎么实现功能。这样的写法显得有点过时了，我们进行下简单的改造：

```js
[1,2,3,4,5,6].map(num => Math.pow(num,2))
```

使用map函数，代码一方面逻辑简化了不少，另一方面也引入了函数式编程。

结合上面的例子我们先总结一下：

> **命令式编程**具体告诉计算机怎么干活。
> 
> **声明式编程**是将程序的描述与求值分离，关注如何用表达式描述程序逻辑。

我们再看一个对变量进行加一例子，引入函数式编程的第二个特性：

```js
let counter = 0;

function increment(){
    return ++counter
}
```

这个increate方法是有一定的缺陷，你也许发现了，该方法对其他作用域有副作用(side-effect)，影响到其他作用域变量的值。所以得进行适当得改造。

```js
let increment = counter => ++counter
```

改造后，函数内修改能影响到的也只是传入的参数，避免了副作用。

需要知道 的是，改造后代increate函数是一个纯函数，纯函数有以下个性：

1、函数返回值仅取决于输入参数。
2、不能造成超出作用域外的状态变化。

以上两个例子介绍了函数式编程的两个基本特性：声明式、纯函数行。函数式编程旨在帮你编写干净的、模块化的、可测试的并且简洁的代码，提高代码的无状态性和不变性。遵循以下原则：

   声明式(Declarative)
   纯函数(Pure Function)
   数据不可变性(Immutablity)

##### 4.5.2 函数式编程的好处

函数式编程到底有什么好处呢，为什么变得越来越流行？总结一下有这几个好处：

1、把任务分解成简单的函数。

假设我们要在页面上根据学号显示学生的姓名、年级。拿到题目后，脑子里立刻产生的代码可能是这样的:

```js
function showStudent(stuNum){
  //stub,假设有这么一个store，可以根据学号获得学生对象。
  var student = store.get(stuNum);
  if(student){
    document.querySelector("${target}").innerHTML = '${student.name},${student.grade}'
  }
}
showStudent（'011526'）
```

这个函数从功能上看是没有问题的，能够满足功能要求。但还是有几个问题比较突出，第一就是代码是强耦合的，测试变得变得比较复杂；第二是代码无法复用。这与函数编程的组合思想相冲突，所以，还是得先给它动个"手术"，

```js
var find = curry(function(store,stuNum){
  var student = store.get(stuNum);
  if(student != null){
    return student;
  }
});

var objToString = function(student){
  return `${student.name},${student.grade}`
}

var append = curry(function(target,info){
  document.querySelector(target).innerHTML = info;
});

var showStudent= compose(append("target_id", objToString,find(store)));

showStudent('222')
```

这时候你可能会想，Curry函数和compose函数是张的什么样子呢，先不用着急。我们在稍后介绍的函数式编程基础部分就详细介绍这两个方法。上面"手术后"的代码变化较大，由刚开始的一个函数变成了4个，各个函数的职责也更加清楚，find方法负责从持久化对象中获得学生对象，objToString方法负责输出学生信息，append方法负责把信息添加到目标对象上，compose方法是负责把各个方法组合起来。

> 友情提示：职责单一不仅是函数式编程的重要特点，也是软件设计SOLID中非常重要的一个原则。

2、接近自然语言，方便理解

函数式编程的自由度比较高，可以实现接近自然语言的代码，比如我们要计算一个数学表达式 (2+4)*15+ 72的值，翻译成代码语句可能要编程这样的：

```js
add(multiply(sum(2,4), 15))
```

这样的实现方式是不是更容易理解？

3、更容易的代码管理

函数式编程的原则之一是纯函数，状态不受外部因素影响，也不会改变外部状态。只要是传入的参数相同，得到的结果也一定是相同的（幂等）。所以每一个函数都是独立的，测试也变得很轻松。

##### 4.5.3 函数式编程基础

上面看了函数式编程的诸多好处，那么我们怎么才能编写出好的函数式代码呢，下面就这个问题我们详细看下函数式编程的基础。

1、高阶函数（HOF,higher-order function）

在数学和计算机科学中，高阶函数至少要满足下列一个条件：

> 1、接受一个或多个函数作为输入
> 
> 2、返回一个函数

我们先写一个过滤器的高阶函数：

```js
const filter = (predicate, xs) => xs.filter(predicate)
```

再增加一个类型判断辅助函数：

```js

const is = (type) => (x) => Object(x) instanceof type
```

测试一下：

```js
filter(is(Number), [0, '1', 2, null,"name","33"])
```

输出[0, 2]，这是我们想要的结果。

2、柯里化(Currying)

 柯里化是将多个参数的函数(多元函数) 转换为 一元函数的过程。每当函数被调用时，它仅仅接收一个参数并且返回带有一个参数的函数，直到所有的参数传递完成。

我们先定义一个柯里化求值函数 curriedSum，

```js
 const curriedSum = (a) => (b) => a + b
```

我们简单看下这个函数的执行过程，第一次调用：

```
curriedSum(16)
```

返回

```js
(b) => 16 + b
```

继续调用

```js
curriedSum(16)(24)
```

最后得到结果：40

3、闭包(Closure)

闭包 是指有权访问另一个函数作用域中的变量的函数。闭包形成的最常见的方式就是在一个函数内创建另一个函数，通过另一个函数访问这个函数的局部变量。

总体来说，闭包有3个特性：  
（1）函数嵌套  
（2）内部函数可以引用外部函数的参数、变量和函数表达式  
（3）闭包不会被垃圾回收机制回收，常驻内存

先声明一个简单的闭包形式：

```js
function init() {
  var name = '首席coder'; 
  var innerCount = 0;
  function displayName() { 
    innerCount++;
    alert(name);    
  }
  function getCount(){
    alert(innerCount)
  }
  displayName(); 
  displayName();   
  getCount(); //2
}
```

方法displayName声明在init方法内形成闭包。通过两次调用displayName方法将innerCount增加，通过显示的值可以看出，闭包涉及的变量会常驻内存，不会被释放。所以说闭包数量过多会导致内存泄漏。

4、函数组合(Function Composition)

函数组合是函数式编程中很重要的组成部分，函数式编程崇尚细粒度的函数实现，细粒度实现后，然后呢？怎么把这些函数组织起来有效的工作？

这就是函数的组合功能，先看下函数组合的雏形吧：

```js
const compose = (f, g) => (a) => f(g(a)) 
```

compose中接收两个函数(为了简单起见，我们不对参数进行类型校验，默认为函数类型)作为参数，对传进来的参数先有g函数进行运算，再把运算结果作为f函数的入参继续运算。

有了组合函数，我们还需要定义规则，说明要一个什么样的功能，比如说把数字四舍五入并转换为字符串，这里需要两个方法，一个是Math的round方法，一个是toString方法，有了“作料”就可以“开火”了。把这两个整合成一个方法floorAndToString 

```js
const floorAndToString = compose((val) => val.toString(), Math.round)
```

等不及了，还是先测试下：

```js
floorAndToString(121.212121) // '121'
floorAndToString(121.512121) // '122'
```

顺利通过测试。到现在函数式编程最重要的原则都已介绍，回头再看4.5.2中优化后的代码，是不是简单多了，也比较容易理解了。

5、其他原则

函数式编程还有其他的原则，由于篇幅问题就不做详细介绍了，在这里这是简单列举各个原则，各位读者可以自行查阅:

    幂等 (Idempotent)

    偏应用函数 (Partial Application)

    断言 (Predicate)

    自动柯里化 (Auto Currying)

> 附上笔者整理的帖子：http://www.houyuewei.cn/2018/08/23/js-func-program-term

#### 

#### 4.6 实践：状态原理解析

 前端技术的发展日新月异，开发社区里流行这么一句话“前端圈没三个月就会有新的技术出现”，虽说有点夸张，但是从侧面说明了前端的变化之快。前端也经过几年的沉淀，逐渐形成了React，Vue，angular为领袖的三大开发框架和各自的全家桶，带来了全新的开发体验。

随着页面复杂度的升级，对应的localstorage，vuex，redux，mobx等数据存储和管理方案也渐渐复现。所以对状态管理的原理进行一定的了解还是很有必要的，了解基本原理也方便理解其他的管理库。

现在我们以一个简单的例子，一步一步来解析下state到底是怎么回事，具体是怎么工作的？

先看下基本的原理图：

![](images/state-1.png)

我们已完整的单向数据流模型进行说明，数据流向有单向(如vuex、redux,)和双向(mobx)之分，和双向相比单向数据流更具有可维护性的特点，所以以此模型进行说明。

事件默认都是从UI页面进行发起，dispatch一个action，也就是说在单数据流的模型中，状态的改变都是以触发action作为入口条件，由action中commit一个mutation更新state中的数据，状态改变自动更新页面。



阐述完状态的基本原理后，我们计划实现一个这样的页面（数据不持久化）

![](images/state-2.png)

已完成任务部分(List组件)和右侧完成任务(count组件)情况分别对应两个组件。



第一步先看下View组件，先定义一个component的基类，

```js
export default class Component {
  constructor(props = {}) {
  
    // 继承该类的组件应该实现该方法，用来渲染组件
    this.render = this.render || function() {};

    if (props.store instanceof Store) {
      props.store.events.subscribe("stateChange", () => self.render());
    }

    //如果element元素，就把改元素设置为元素挂载节点
    if (props.hasOwnProperty("element")) {
      this.element = props.element;
    }
  }
}
```

我们约定各子组件要实现render方法，在render方法中实现DOM结构。this.element指定DOM结构的挂载位置。为了实现页面的自动更新，子组件借助发布/订阅模式订阅stateChange事件。store中当state中的数据更新时，会发布该事件，子组件收到通知后重新渲染。该行为类似React中的setState中的效果。

再看一下组件的具体实现，Count.js：

```js
import Component from "../lib/component.js";
import store from "../store/index.js";
import _ from "../lib/utils.js";

export default class Count extends Component {
  constructor() {
    super({
      store,
      element: _.$(".js-count") //获得dom节点
    });
  }

  render() {
    let emoji = store.state.items.length > 0 ? "🙌" : "😢";

    this.element.innerHTML = `
            <small>你今天已完成</small>
            <span>${store.state.items.length}</span>
            <small>条任务 ${emoji}</small>
        `;
  }
}
```

子组件中通过调用父组件的构造函数完成事件订阅。

```js
export default class PubSub {
  constructor() {
    this.events = {};
  }

  /**
   * 订阅事件，并注册回调方法
   */
  subscribe(event, callback) {
    let self = this;

    if (!self.events.hasOwnProperty(event)) {
      self.events[event] = [];
    }

    return self.events[event].push(callback);
  }
}
```

下面梳理下store的情况，store由action,mutation和state三部分组成，acation用来标识每个请求，也是触发state变化的唯一因素。mutation类似于事件，每个mutation都有一个事件类型和回调函数，这个回调函数是进行状态改变的地方，接收state和payload作为参数。

```js
export default new Store({
    actions,
    mutations,
    state
});
```

store.js

```js
export default class Store {
  constructor(params) {
    let self = this;
    //定义actions,mutations和state，
    //在初始化中，需要把action和mutation都初始化进来
    self.actions = {};
    self.mutations = {};
    self.state = {};

    // A status enum to set during actions and mutations
    self.status = "resting";

    // 初始化发布-订阅模型
    self.events = new PubSub();

    //如果传入actions，就使用传入的actions
    if (params.hasOwnProperty("actions")) {
      self.actions = params.actions;
    }

    if (params.hasOwnProperty("mutations")) {
      self.mutations = params.mutations;
    }

    //对state的值设置拦截
    self.state = new Proxy(params.state || {}, {
      set: function(state, key, value) {
        state[key] = value;
        // 发布 stateChange通知
        self.events.publish("stateChange", self.state);

        return true;
      }
    });
  }
  }
```

把store中的state设置成全局state数据模型的代理，为什么要这么做呢？因为前面已经提高，我们要让state扮演成状态机的角色，state变引起页面的渲染，此时Proxy倒是一个很好的选择。

> 对Proxy还不是很熟悉 的同学可以参考https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy



我们看Proxy的set钩子里，state变化，就会发布stateChange通知各个子组件。



store又是怎么分发action的呢？我们来探究一下，看下store中定义的dispatch函数，接受type类型和数据对象

```js
dispatch(actionKey, payload) {

    // 校验action是否存在
    if (typeof self.actions[actionKey] !== "function") {
      console.error(`Action "${actionKey} doesn't exist.`);
      return false;
    }

    // 分组显示action 信息
    console.groupCollapsed(`ACTION: ${actionKey}`);

    // 设置action，说明我们正在dispatch一个action
    self.status = "action";

    //调用action
    self.actions[actionKey](self, payload);

    // Close our console group to keep things nice and neat
    console.groupEnd();

    return true;
  }
```

在view组件中，通过

```js
store.dispatch(types.ADDITEM, value);
```

派发action，需要为每个action指定类型，使用该字段用来区别是什么类型的action。所有的action因为在store初始化的时候已经注入，所以只需要根据action type来判断对应的action是否存在。

action.js

```js
export default {
  addItem(context, payload) {
    context.commit(types.ADDITEM, payload);
  },
  clearItem(context, payload) {
    context.commit(types.CLEARITEM, payload);
  }
};

```

如果找到对应的action，那么立即执行。我们注意到

```js
self.actions[actionKey](self, payload);
```

action的执行需要把self传进来，所以action中方法的第一个参数还是指向store。所以store继续commit

```js
commit(mutationKey, payload) {

    // 校验mutation是否存在
    if (typeof .mutations[mutationKey] !== "function") {
      console.log(`Mutation "${mutationKey}" doesn't exist`);
      return false;
    }
    ...
    // 创建一个新的state，并将新的值附在state上
    let newState = this.mutations[mutationKey](this.state, payload);

    // 替换旧的state值
    this.state = Object.assign(this.state, newState);

    return true;
  }
```

对应commit中mutationKey，是从action顺延下来的。和action的处理过程相似，对应的mutation也是初始化加载，需要根据key值处理对应的mutation。

mutation.js

```js
export default {
    addItem(state, payload) {
        state.items.push(payload);
        
        return state;
    },
    clearItem(state, payload) {
        state.items.splice(payload.index, 1);
        
        return state;
    }
};

```

来了，来了，它来了。state带着口罩风风火火的来了。对应addItem类型的mutation，在state数组中push一条记录。到这里，你可能也已恍然大悟，state的状态机原来是这么工作的，Proxy的set钩子原来是这么被触发的，子组件原来是在这种情况下重新工作的。

一切都变得顺理成章了。现在可以启动下示例代码或者按照这个思路重新实现一遍，看看效果是怎么样的。








