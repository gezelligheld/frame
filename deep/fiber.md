react 16版本之前，当一次状态变化需要更新时，会从根节点递归更新，随着项目的复杂度提升，嵌套的层级越来越多，难免会出现卡顿的情况

为了解决这样的问题，react 16版本引入了fiber架构，将之前的 stack reconciler 重构成新版的 fiber reconciler，变成了具有链表和指针的 单链表树遍历算法。通过指针映射，每个单元都记录着遍历当下的上一步与下一步，从而使遍历变得可以被暂停和重启，是一种任务分割调度算法

fiber是react中最小粒度的执行单元，更新fiber的过程称为调和（Reconciler），每个fiber可以根据自身的优先级判断是否还有空闲时间去执行更新，如果没有会将主线程让出，让浏览器执行一些渲染工作，避免卡顿；等到浏览器有了空闲时间，再通过调度器scheduler恢复之前的执行单元，继续更新任务

#### 概念

jsx最终会被创建出element对象，每个类型的element对象都有一个与之对应的fiber类型，element变化引起的更新流程都是在fiber层面去做的，如果是元素类型的话，会形成新的dom进行视图渲染

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a90368f24f0477aaf0d446a8f6736db~tplv-k3u1fbpfcp-watermark.awebp)

element 与 fiber 之间的对应关系如下

```js
export const FunctionComponent = 0;       // 对应函数组件
export const ClassComponent = 1;          // 对应的类组件
export const IndeterminateComponent = 2;  // 初始化的时候不知道是函数组件还是类组件 
export const HostRoot = 3;                // Root Fiber 可以理解为跟元素 ， 通过reactDom.render()产生的根元素
export const HostPortal = 4;              // 对应  ReactDOM.createPortal 产生的 Portal 
export const HostComponent = 5;           // dom 元素 比如 <div>
export const HostText = 6;                // 文本节点
export const Fragment = 7;                // 对应 <React.Fragment> 
export const Mode = 8;                    // 对应 <React.StrictMode>   
export const ContextConsumer = 9;         // 对应 <Context.Consumer>
export const ContextProvider = 10;        // 对应 <Context.Provider>
export const ForwardRef = 11;             // 对应 React.ForwardRef
export const Profiler = 12;               // 对应 <Profiler/ >
export const SuspenseComponent = 13;      // 对应 <Suspense>
export const MemoComponent = 14;          // 对应 React.memo 返回的组件
```

一个fiber节点上会保存这些信息

```js
function FiberNode(){

  this.tag = tag;                  // fiber 标签 证明是什么类型fiber。
  this.key = key;                  // key调和子节点时候用到。 
  this.type = null;                // dom元素是对应的元素类型，比如div，组件指向组件对应的类或者函数。  
  this.stateNode = null;           // 指向对应的真实dom元素，类组件指向组件实例，可以被ref获取。
 
  this.return = null;              // 指向父级fiber
  this.child = null;               // 指向子级fiber
  this.sibling = null;             // 指向兄弟fiber 
  this.index = 0;                  // 索引

  this.ref = null;                 // ref指向，ref函数，或者ref对象。

  this.pendingProps = pendingProps;// 在一次更新中，代表element创建
  this.memoizedProps = null;       // 记录上一次更新完毕后的props
  this.updateQueue = null;         // 类组件存放setState更新队列，函数组件存放
  this.memoizedState = null;       // 类组件保存state信息，函数组件保存hooks信息，dom元素为null
  this.dependencies = null;        // context或是时间的依赖项

  this.mode = mode;                //描述fiber树的模式，比如 ConcurrentMode 模式

  this.effectTag = NoEffect;       // effect标签，用于收集effectList
  this.nextEffect = null;          // 指向下一个effect

  this.firstEffect = null;         // 第一个effect
  this.lastEffect = null;          // 最后一个effect

  this.expirationTime = NoWork;    // 通过不同过期时间，判断任务是否过期， 在v17版本用lane表示。

}
```

fiber节点间通过return、child、sibling属性相互关联，如下面的代码及对应的fiber树

```js
export default class Index extends React.Component{
   state={ number:666 } 
   handleClick=()=>{
     this.setState({
         number:this.state.number + 1
     })
   }
   render(){
     return <div>
       hello，world
       <p > 《React进阶实践指南》 { this.state.number } 👍  </p>
       <button onClick={ this.handleClick } >点赞</button>
     </div>
   }
}
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bdf7dc554e54197a98bbc9be5b191b2~tplv-k3u1fbpfcp-watermark.awebp)

#### fiber更新机制

1. 初始化

首次构建应用， 创建一个 fiberRoot ，作为整个 React 应用的根基

通过 ReactDOM.render 渲染出来的，可以作为一个 rootFiber。一个 React 应用可以有多 ReactDOM.render 创建的 rootFiber ，但是只能有一个 fiberRoot

第一次挂载后，fiberRoot和rootFiber建立联系

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb68640d39914c03bc77ea15616c7918~tplv-k3u1fbpfcp-watermark.awebp)

正在内存中构建的 Fiber 树称为 workInProgress Fiber 树，一次更新中，所有的更新都是发生在 workInProgress 树上。在一次更新之后，workInProgress 树上的状态是最新的状态，那么它将变成 current 树用于渲染视图

渲染rootFiber时，会复用当前current 树的alternate作为workInProgress树，如果没有则创建一个fiber作为workInProgress

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a7f5a9b77ff45febd8e255fcba1ba3a~tplv-k3u1fbpfcp-watermark.awebp)

然后在workInProgress树上完成fiber的创建及遍历

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cda0522c0c85435494ccf3a3ea587baa~tplv-k3u1fbpfcp-watermark.awebp)

最终以workInProgress树作为current渲染树，完成初始化的流程

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/907fb4f6768842438e0df7f083adc4b6~tplv-k3u1fbpfcp-watermark.awebp)

2. 更新

当发生状态更新时，首先复用current树上的alternate，创建一个新的workInProgress树，其子节点也需要与current树上的子节点通过alternate建立关联，最终还是以最新的workInProgress树作为current树进行渲染

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ff00ce5f2db0430c841ea3a01754542e~tplv-k3u1fbpfcp-watermark.awebp)

一个在内存中构建，一个渲染视图，两颗树用 alternate 指针相互指向，在下一次渲染的时候，直接复用缓存树做为下一次渲染树，上一次的渲染树又作为缓存树，这样可以防止只用一颗树更新状态的丢失的情况，又加快了 DOM 节点的替换与更新，称之为双缓冲树

#### reconciler调和

fiber更新流程分为render阶段和commit阶段

##### render阶段

首先workloop会遍历一遍fiber树

```js
// react-reconciler/src/ReactFiberWorkLoop.js
function workLoop (){
    while (workInProgress !== null ) {
      workInProgress = performUnitOfWork(workInProgress);
    }
}

function performUnitOfWork(){
    next = beginWork(current, unitOfWork, renderExpirationTime);
    if (next === null) {
       next = completeUnitOfWork(unitOfWork);
    }
}
```

其中performUnitOfWork主要做了两个事情

1. beginWork

向下调和的过程，由fiberRoot按照child指针逐层向下调和，期间会执行函数组件，实例类组件，及其部分生命周期，执行render，得到最新的children；diff 调和子节点；打上不同副作用标签effectTag

```js
// react-reconciler/src/ReactFiberBeginWork.js
function beginWork(current,workInProgress){

    switch(workInProgress.tag){
       case IndeterminateComponent:{// 初始化的时候不知道是函数组件还是类组件 
           //....
       }
       case FunctionComponent: {//对应函数组件
           //....
       }
       case ClassComponent:{  //类组件
           //...
       }
       case HostComponent:{
           //...  
       }
       ...
    }
}

function reconcileChildren(current,workInProgress){
   if(current === null){  /* 初始化子代fiber  */
        workInProgress.child = mountChildFibers(workInProgress,null,nextChildren,renderExpirationTime)
   }else{  /* 更新流程，diff children将在这里进行。 */
        workInProgress.child = reconcileChildFibers(workInProgress,current.child,nextChildren,renderExpirationTime)
   }
}
```

以下是常见的effectTag

```js
export const Placement = /*             */ 0b0000000000010;  // 插入节点
export const Update = /*                */ 0b0000000000100;  // 更新fiber
export const Deletion = /*              */ 0b0000000001000;  // 删除fiebr
export const Snapshot = /*              */ 0b0000100000000;  // 快照
export const Passive = /*               */ 0b0001000000000;  // useEffect的副作用
export const Callback = /*              */ 0b0000000100000;  // setState的 callback
export const Ref = /*                   */ 0b0000010000000;  // ref
```

2. completeUnitOfWork

向上归并的过程，有兄弟节点返回兄弟节点，没有返回父级节点，直到fiberRoot

首先 completeUnitOfWork 会将 effectTag 的 Fiber 节点会被保存在一条被称为 effectList 的单向链表中，在 commit 阶段将不再需要遍历每一个 fiber ，只需要执行更新 effectList 就可以了；会进行事件收集，最终统一交给根节点处理；会创建真实dom

##### commit阶段

这个阶段会进行componentDidMount、函数组件useEffect、useLayoutEffect的处理，还会进行节点的增删改

1. Before mutation

执行DOM操作前，此时DOM还没有被修改，可以获取DOM快照，getSnapshotBeforeUpdate生命周期会在这个阶段执行；异步调用了useEffect，不阻塞视图渲染

2. Mutation

执行DOM操作，进行DOM节点的增删改，同时置空了ref（方便垃圾回收）

3. Layout

执行DOM操作后，会执行类组件setState 的callback，以及函数组件的useLayoutEffect（会阻塞视图渲染）