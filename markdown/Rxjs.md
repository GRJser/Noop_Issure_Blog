# Rx.js 笔记

### 安装

首先rxjs的不同版本的Api差距很大，v4和v5的基本实现方式不一样，所以我们需要区分。

```shell
npm i rxjs        ##v5版本
npm i rx          ##v4版本
```

```javascript
import Rx from 'rxjs/Rx';
import Rx from 'rxjs';

//深链
import { Observable } from 'rxjs/Observable';
```

上面的导入方式是将 Rx 整个库全部导入进来。有的时候，我们一个项目并不需要一整个Rx的全部功能，为了防止打包后体积过大，我们可以采取 **<u>深链</u>** 的方式来应用Rx。

> 关于TreeSharking对Rx起不了太大的作用，因为Rx在底层Observale这个类几乎引用了所有的代码

### 观察者模式

![image-20190823113047792](https://github.com/GRJser/Noop_Issure_Blog/blob/master/image/image-20190823113047792.png?raw=true)

Observable （可以被观察的对象）

> 可以看成发布者（不是生产数据的生产者，看Hot章）

Observer（观察者）                

> 连接这两个的桥梁就是（subscribe）

```javascript
import { Observable } from 'rxjs/Observable';
import 'rxjs/add/observable/of';
//使用分开引入的方式添加方法

//source就是可被观察者 console.log扮演了观察者	也可以看到subscribe充当一个桥梁
const source = Observable.of(1, 2, 3);  
source.subscribe(console.log);          
```

### 迭代器模式

提到迭代器，在js中的迭代器，我们是调用next函数来一个一个吐出数据的模式。

实现迭代器，也就是（游标），遍历一个数组。



但是无论怎么实现这个设计，一般都有几个通用的Api

- getCurrent 获取当前 指针指向的元素
- moveToNext 将 指针移到下一项元素 
- isDone 判断是否已经遍历完一个集合

在Rxjs 中我们不需要去（拉）（调用getCurrent）数据。我们只要通过subScribe连接上Observeable后，就可以收到消息，这就是（推）

### Observable简单实例

```javascript
const onSubscribe = observer => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
};
const source$ = new Observable(onSubscribe);
const theObserver = {
  next: item => console.log(item)
};
source$.subscribe(theObserver);
```

1. 创建一个onSubscribe函数，他会作为Observable的参数，决定了Observable的行为，当调用next的时候，将数据传给Observer。
2. 调用Observable构造函数，生成一个source$数据流（由于rxjs推数据的形式就像一个流）
3. 创建观察者Observer。
4. 通过subscribe将theObserver和source$关联起来

在函数底层：

1. 定义了onSubscribe函数，这个函数作为参数传递给Observable构造函数，创建了一个source$对象。 

   `这个时候，onSubscribe并没有被调用，他只是在等待source$的subscribe被调用）`

2. theObserver对象被创建。

3. source的subscribe方法被调用，theObserver成为了source​的观察者。

   `每有一个subsrcribe调用，都会重新运行onSubscribe函数。`

   `就在调用subscribe的时候，onSubscribe函数才会被调用，在运行中函数的参数observer代表的就是观察者theObserver，但并不是theObserver对象本身，在这里，rxjs实际上会对观察者做一个包装，在这里参数observer就是被包装的theObserver，所以二者完全不一样，可以理解为参数observer代理了theObserver，就像vue的data，methods等一样，对参数observer的所有函数调用都会转移到theObserver的同名函数上去`

4. 在onSubscribe函数中，连续调用了三次next方法，通过observer的代理，也就是调用了theObserver的next方法被调用了三次，所以console.log运行了三次，打印出三个数字

### Observable异步推送和关闭

```javascript
const onSubscribe = observer => {
  let number = 1;
  const handle = setInterval(() => {
    observer.next(number++);
    if (number > 6) {
      clearInterval(handle);
    }
  }, 1000);
};
```

想要异步推送当然只需要异步调用observer的next方法就可以了，理所当然的逻辑。

之所以可以这样做，是因为observer参数完全代理了观察者对象theObserver，对于观察者来说，他不需要操心任何事情，只需要被动的接受数据来处理即可，也不必担心数据什么时候产生。

~~***调用next方法的操作有时被称为emit***~~

如果我们不清除定时器（clearInterval），那么观察者会一直接受到数据，Observable可以推送的数据是∞的，但是这个程序并不会消耗更多的内存（不会再⬆️）这是因为Observable对象每次只吐出一个数据，然后这个数据就被Observable消化处理了，不会存在数据的堆积。

*⚠️这种数据的处理方式，和吧数据都堆积在一个数组里，然后挨个处理数据有很大的差别。*

如果我们关闭了定时器，不再继续推送数据，那么这个数据流理应结束，因为观察者不会在收到数据，当实际上并没有，我们需要手动给予Observer一个信号，告诉他Observable对象已经完结的信号。

```javascript
//我们在theObserver对象上添加了一个complete的方法，代表没有更多的数据会被传入。
const theObserver = {
  next:item => console.log(item),
  complete:() => console.log('the end')
}
//我们需要用参数observer代理的方式来调用complete方法
const onSubscribe = observer => {
  let number = 1;
  const handle = setInterval(() => {
    observer.next(number++);
    if (number > 6) {
      observer.complete();
      clearInterval(handle);
    }
  }, 1000);
};
```

### Observable的出错处理

```javascript
//除了告诉推送给观察者  数据/结束信号之外，我们还要给观察者一个出错信号。
const onSubscribe = observer => {
  observer.next(1);
  observer.error('404 not found')
  observer.complete();
};

const theObserver = {
  next: item => console.log(item),
  complete: () => console.log('the end'),
  error: err => console.log(err)
}
```

如果运行代码就会发现。complete函数没有运行。

在Rx中，一个Observable对象只有一个终结状态，要么complete，要么error!

![image-20190823114749303](Rxjs.assets/image-20190823114749303.png)

⚠️之前提到observer只是代理theObserver，所以调用complete函数并没啥卵用。

### Observer简化

对于theObserver对象来说，如果complete的时候不需要做一些特殊的操作的话，那么可以不提供complete方法，比如不会结束的Observable，complete函数毫无用处。

如果没有提供error方法却出错了，那么这个错误会一直往外抛，所以error方法有比较👌。

对于subscribe函数来说，也可以接受的不是一个对象，可以直接接受3个函数

```javascript
source$.subscribe(theObserver)

source$.subscribe(
  item => console.log(item),
  err => console.log(err),  //null
  () => console.log('the end')
)
```

如果不关心某个处理函数，比如err函数，可以直接把函数替换成null

### 退订Observabe

有观察，有退订是理所当然的事情。

我们通过subscribe建立Observer和Observable的联系，要通过unsubscribe退订。

```javascript
const onSubscibe = observer => {
  let count = 1
  const time = setInterval(() => {
    observer.next(count);
    count++
  },1000);
  return {
    unsubscribe: () => {
      clearInterval(time)
    }
  }
}
```

可以看到onSubscibe函数返回了一个对象，其中包含了一个unsubscribe方法，只要调用这个方法，就可以清除定时器，来退订。

### Hot/Cold  Observable

如果一个Observable被二个Observer订阅。但是不是同时订阅，一号比二号早5秒，那么对于二号来说是不是那5秒的数据丢失了呢。

对于不同的需求场景，我们希望有不同的结果。

1. 没丢失，比如支付宝的通知，你不能丢失吧。（Cold）

2. 丢失了，比如直播，错过也就丢失了吧。（Hot）

对于Cold来说，没丢失就相当于每次订阅都会产生一个对应的推送中心（每次订阅都会产生闭包的感觉）

```javascript
const cold$ = new Observable( observer => {
  const producer = new Producer();
  //然后然observer去接受producer产生的数据
})
```

对于Hot来说，概念上有个独立的生产者，他的创建和subscribe的调用没有关系，sub的调用只是然Observer连接上生产者。

```javascript
const producer = new Producer();
const Hot$ = new Observable( observer => {
  //然后然observer去接受producer产生的数据
})
```

所以想要HOT还是COLD都不是通过控制Observable来实现的，实现的方法只有通过生产者自身来控制，他是HOT还是COLD就决定Observable是HOT还是COLD

### 操作符

![image-20190823115212919](https://github.com/GRJser/Noop_Issure_Blog/blob/master/image/image-20190823115212919.png?raw=true)

在实际运用Rx的时候，往往需要对数据进行一些处理，也就是操作。

对于每个操作符来说，连接的就是上游和下游，可以想象Node的pipe函数，只不过有点不同。

简单的实例：

```javascript
import { Observable } from 'rxjs/Observable';
import 'rxjs/add/operator/map';
const onSunsrcibe = observer => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
};
const source$ = Observable.create(onSunsrcibe);
source$.map(x => x ** 2).subscribe(console.log);
```

可以看到创建发布者的创建不自由通过New一个的方式来创建。

第一个操作符是Observable自带的create，意思是创建一个Observable对象。

第二个就是map，对于js来说这玩意太熟悉了，就是差不多的意思。

*⚠️（只不过生产的不是数组了， 而是一个新的Observable对象  ）*

总而言之，我们可以把操作符看成--reducer

如果要给一个定义：

*一个操作符是返回一个Observable对象的函数，不过，有的操作符是根据其他Observable对象产生返回的Observable对象，有的是利用其他类型的输入产生返回的Observable对象，还有一些操作符不需要输入就可以凭空创造一个Observable对象。*

### 操作符和弹珠图

对于rxjs数据流的形式简单来说就是单项的数据流，稍微复杂的就需要使用弹珠图来辅助。

![image-20190823115330739](https://raw.githubusercontent.com/GRJser/Noop_Issure_Blog/master/image/image-20190823115330739.png)

上图就是一个简单的弹珠图，意思就是每间隔一段时间后吐出一个递增的正整数。

根据弹珠图的格式，| 竖杠代表的是数据流的结束。也就是complete函数的调用。x 叉号就是数据流的异常，也就是error函数的调用。所以这⚠️二个符号不可能同时存在

![image-20190823115349569](https://raw.githubusercontent.com/GRJser/Noop_Issure_Blog/master/image/image-20190823115349569.png)

为了表达操作符，就会用上下游的方式来表达，如下：

![image-20190823115404539](https://raw.githubusercontent.com/GRJser/Noop_Issure_Blog/master/image/image-20190823115404539.png)

根据功能可分为：

1. 创建类（例如create）
2. 转化类 （例如map）
3. 过滤类（例如filter）
4. 合并类
5. 多播类
6. 错误转化类
7. 辅助工具类
8. 条件分支类
9. 数学和合计类
10. 背压控制类
11. 可连接类
12. 高阶Observable处理类

从存在形式来分。（ 静态） 和（ 实例）分类

🌰 Observable.of是静态方法，Observable.prototype.map是实例方法

在导入的时候也是有区别的：

```javascript
import  'rxjs/add/observable/of'
import  'rxjs/add/operator/map'
```

当然，无论是什么种类，他都应该返回一个Observable对象。

每个操作符都必须考虑几件事：

- 返回一个全新的Observable对象
- 对上下游的订阅和退订处理
- 处理异常
- 及时释放资源

