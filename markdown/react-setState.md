# setState异步/同步

关于this.setState是异步还是同步的问题困扰了很久。

在React中，如果是由React引发的事件处理（比如通过onClick引发的事件处理），调用setState不会同步更新this.state，除此之外的setState调用会同步执行this.state。所谓“除此之外”，指的是绕过React通过addEventListener直接添加的事件处理函数，还有通过setTimeout/setInterval产生的异步调用。

```react
// setState不会立刻改变React组件中state的值；
const currentCount = this.state.count;
this.setState({count: currentCount + 1});
this.setState({count: currentCount + 1});
this.setState({count: currentCount + 1});
console.log(this.state.count);
// 多次setState函数调用产生的结果会合并
this.setState({FirstName: 'Morgan'});
this.setState({LastName: 'Cheng'});
// 连续调用了两次this.setState，但是只会引发一次更新生命周期
// 不是两次，因为React会将多个this.setState产生的修改放在一个队列里
// 缓一缓，攒在一起，觉得差不多了再引发一次更新过程。
// 相当于如下：
this.setState({FirstName: 'Morgan', LastName: 'Cheng'});
```

setState通过引发一次组件的更新过程来引发重新绘制,他的调用引发了生命周期的4个函数的调用（依次调用）

1. shouldComponentUpdate. `他调用的时候this.state并没有更新,如果返回false，更新中断，但是this.state还是会更新*`
2. componentWillUpdate   `他调用的时候this.state也没有更新`
3. render  `这时候this.state才更新`
4. componentDidUpdate

从上面的顺序来看，比props的变更少一个componentWillReceiveProps函数的调用



为了实现合并等功能，他就是要异步调用更新，但是setState可以传递一个函数

```react
// 第一个是当前的state值，第二个是当前的props
// 每次函数运行时接到的state是同一个对象，但他不是真正的state
function increment(state, props) {
  console.log('后运行'）
  return {count: state.count + 1};
}
this.setState(increment);
this.setState(increment);
this.setState(increment);
console.log('先运行'）
            
            
            
// 2种我都用
this.setState(increment);
this.setState(increment);
this.setState({count: this.state.count + 1});
this.setState(increment);
// 结果是 2 不是 4，因为半路杀出的传统式调用导致函数式的调用效果清空。
```

虽然函数异步的更新state，但是我们可以写出这种更新方式