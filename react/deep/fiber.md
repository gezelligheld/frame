react 16 版本之前，当一次状态变化需要更新时，会从根节点递归更新，随着项目的复杂度提升，嵌套的层级越来越多，难免会出现卡顿的情况

为了解决这样的问题，react 16 版本引入了 fiber 架构，将之前的 stack reconciler 重构成新版的 fiber reconciler，变成了具有链表和指针的 单链表树遍历算法。通过指针映射，每个单元都记录着遍历当下的上一步与下一步，从而使遍历变得可以被暂停和重启，是一种任务分割调度算法

fiber 是 react 中最小粒度的执行单元，每个 fiber 可以根据自身的优先级判断是否还有空闲时间去执行更新，如果没有会将主线程让出，让浏览器执行一些渲染工作，避免卡顿；等到浏览器有了空闲时间，再通过调度器 scheduler 恢复之前的执行单元，继续更新任务

#### 概念

jsx 最终会被创建出 element 对象，每个类型的 element 对象都有一个与之对应的 fiber 类型，element 变化引起的更新流程都是在 fiber 层面去做的，如果是元素类型的话，会形成新的 dom 进行视图渲染

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a90368f24f0477aaf0d446a8f6736db~tplv-k3u1fbpfcp-watermark.awebp)

element 与 fiber 之间的对应关系如下

```js
export const FunctionComponent = 0; // 对应函数组件
export const ClassComponent = 1; // 对应的类组件
export const IndeterminateComponent = 2; // 初始化的时候不知道是函数组件还是类组件
export const HostRoot = 3; // Root Fiber 可以理解为跟元素 ， 通过reactDom.render()产生的根元素
export const HostPortal = 4; // 对应  ReactDOM.createPortal 产生的 Portal
export const HostComponent = 5; // dom 元素 比如 <div>
export const HostText = 6; // 文本节点
export const Fragment = 7; // 对应 <React.Fragment>
export const Mode = 8; // 对应 <React.StrictMode>
export const ContextConsumer = 9; // 对应 <Context.Consumer>
export const ContextProvider = 10; // 对应 <Context.Provider>
export const ForwardRef = 11; // 对应 React.ForwardRef
export const Profiler = 12; // 对应 <Profiler/ >
export const SuspenseComponent = 13; // 对应 <Suspense>
export const MemoComponent = 14; // 对应 React.memo 返回的组件
```

一个 fiber 节点上会保存这些信息

```js
function FiberNode() {
  this.tag = tag; // fiber 标签 证明是什么类型fiber。
  this.key = key; // key调和子节点时候用到。
  this.type = null; // dom元素是对应的元素类型，比如div，组件指向组件对应的类或者函数。
  this.stateNode = null; // 指向对应的真实dom元素，类组件指向组件实例，可以被ref获取。

  this.return = null; // 指向父级fiber
  this.child = null; // 指向子级fiber
  this.sibling = null; // 指向兄弟fiber
  this.index = 0; // 索引

  this.ref = null; // ref指向，ref函数，或者ref对象。

  this.pendingProps = pendingProps; // 在一次更新中，代表element创建
  this.memoizedProps = null; // 记录上一次更新完毕后的props
  this.updateQueue = null; // 类组件存放setState更新队列，函数组件存放
  this.memoizedState = null; // 类组件保存state信息，函数组件保存hooks信息，dom元素为null
  this.dependencies = null; // context或是时间的依赖项

  this.mode = mode; //描述fiber树的模式，比如 ConcurrentMode 模式

  this.effectTag = NoEffect; // effect标签，用于收集effectList
  this.nextEffect = null; // 指向下一个effect

  this.firstEffect = null; // 第一个effect
  this.lastEffect = null; // 最后一个effect

  this.expirationTime = NoWork; // 通过不同过期时间，判断任务是否过期， 在v17版本用lane表示。
}
```

fiber 节点间通过 return、child、sibling 属性相互关联，如下面的代码及对应的 fiber 树

```js
export default class Index extends React.Component {
  state = { number: 666 };
  handleClick = () => {
    this.setState({
      number: this.state.number + 1,
    });
  };
  render() {
    return (
      <div>
        hello，world
        <p> 《React进阶实践指南》 {this.state.number} 👍 </p>
        <button onClick={this.handleClick}>点赞</button>
      </div>
    );
  }
}
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bdf7dc554e54197a98bbc9be5b191b2~tplv-k3u1fbpfcp-watermark.awebp)

#### fiber 树的初始化

1. 创建 fiberRoot 和 rootFiber 两个节点

首次构建应用， 创建一个 fiberRoot ，作为整个 React 应用的根基。通过 ReactDOM.render 渲染出来的，可以作为一个 rootFiber。一个 React 应用可以有多 ReactDOM.render 创建的 rootFiber ，但是只能有一个 fiberRoot。第一次挂载后，fiberRoot 和 rootFiber 建立联系

![](https://gitee.com/xiaochen1024/assets/raw/master/assets/20210529105732.png)

2. 根据 jsx 创建 workInProgress Fiber

正在内存中构建的 Fiber 树称为 workInProgress Fiber 树，一次更新中，所有的更新都是发生在 workInProgress 树上。在一次更新之后，workInProgress 树上的状态是最新的状态，那么它将变成 current 树用于渲染视图，current 树表示当前页面中的真实 dom 对应的树，两颗树的节点通过 alternate 相连

渲染 rootFiber 时，会复用当前 current 树的 alternate 作为 workInProgress 树，如果没有则创建一个 fiber 作为 workInProgress 树

![](https://gitee.com/xiaochen1024/assets/raw/master/assets/20210529105735.png)

3. 把 workInProgress Fiber 切换成 current Fiber，完成初始化的流程

![](https://gitee.com/xiaochen1024/assets/raw/master/assets/20210529105738.png)

#### fiber 树的更新

一个在内存中构建，一个渲染视图，两颗树用 alternate 指针相互指向，在下一次渲染的时候，直接复用缓存树做为下一次渲染树，上一次的渲染树又作为缓存树，这样可以防止只用一颗树更新状态的丢失的情况，又加快了 DOM 节点的替换与更新，称之为双缓冲树

1. 根据 current Fiber 创建 workInProgress Fiber

当发生状态更新时，首先复用 current 树上的 alternate，与更新后的 jsx 经过 diff 算法后，创建一个新的 workInProgress 树，其子节点也需要与 current 树上的子节点通过 alternate 建立关联

![](https://gitee.com/xiaochen1024/assets/raw/master/assets/20210529105741.png)

2. 把 workInProgress Fiber 切换成 current Fiber，完成更新流程

![](https://gitee.com/xiaochen1024/assets/raw/master/assets/20210529105745.png)

参考

1. [react 源码解析 7.Fiber 架构](https://xiaochen1024.com/courseware/60b1b2f6cf10a4003b634718/60b1b340cf10a4003b63471f)
