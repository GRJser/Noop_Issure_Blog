# React笔记

关于一个前端框架，对框架最基本的了解应该是它的生命周期，下图是我在查阅过程中总结的react的生命周期示意图。

![image-20190823112017154](Rxjs.assets/image-react生命周期.png)

##### getDerivedStateFormProps(nextProps,prevState)

他可以返回一个state，表示我们需要更新的state，也可以返回一个null，表示不需要更改state。

##### getSnapshotBeforeUpdate(prevProps,prevState)

 他的返回值可以在componentDidMount中作为第三个参数，他的调用时机看上图，Dom还没变。（他可以读取未改变的Dom状态）