更新 fiber 的过程称为调和（Reconciler），其中又分为 render 阶段和 commit 阶段，render 阶段的主要工作是构建 Fiber 树和生成 effectList

首先 workloop 会遍历一遍 fiber 树

```js
// react-reconciler/src/ReactFiberWorkLoop.js
function workLoop() {
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork() {
  next = beginWork(current, unitOfWork, renderExpirationTime);
  if (next === null) {
    next = completeUnitOfWork(unitOfWork);
  }
}
```

其中 performUnitOfWork 会深度优先遍历 fiber 树，有向下遍历和向上回溯的阶段

#### 整体执行流程

向下遍历时，从根节点 rootFiber 开始，遍历到叶子节点，每次遍历到的节点都会执行 beginWork，并且传入当前 Fiber 节点，然后创建或复用它的子 Fiber 节点，并赋值给 workInProgress.child

向上回溯时，每次遍历到的节点都会执行 completeWork 方法，执行完成之后会判断此节点的兄弟节点存不存在，如果存在就会为兄弟节点执行 completeWork，当全部兄弟节点执行完之后，会向上‘冒泡’到父节点执行 completeWork，直到 rootFiber

示例如下

```jsx
function App() {
  return (
    <>
      <h1>
        <p>count</p> xiaochen
      </h1>
    </>
  );
}

ReactDOM.render(<App />, document.getElementById('root'));
```

render 阶段遍历顺序如下，从根节点开始依次执行 beginWork 和 completeWork，最终形成一个 fiber 树，节点间通过 child、return、sibling 相连接

![](https://gitee.com/xiaochen1024/assets/raw/master/assets/20210529105757.png)

#### beginWork

beginWork 主要的工作是创建或复用子 fiber 节点，会根据不同的 tag 创建不同的 fiber。current 树初始化的时候是没有的，所以可以以此来判断是 mount 还是 update

- mount：根据 fiber.tag 进入不同 fiber 的创建函数，最后都会调用到 reconcileChildren 创建子 Fiber

- update：在构建 workInProgress 的时候，当满足条件时，会复用 current Fiber 来进行优化，能复用的条件是

  1. oldProps === newProps && workInProgress.type === current.type 成立，fiber 的 type 不变

  2. 更新的优先级是否足够

```js
// react-reconciler/src/ReactFiberBeginWork.js
function beginWork(current,workInProgress){
    // update时满足条件即可复用current fiber
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false;
      switch (workInProgress.tag) {
        // ...
      }
      // 判断优先级
      return bailoutOnAlreadyFinishedWork(
        current,
        workInProgress,
        renderLanes,
      );
    } else {
      didReceiveUpdate = false;
    }
  } else {
    didReceiveUpdate = false;
  }

  // 根据tag来创建不同的fiber
  switch(workInProgress.tag){
      case IndeterminateComponent:{
          //....
      }
      case FunctionComponent: {
          //....
      }
      case ClassComponent:{
          //...
      }
      case HostComponent:{
          //...
      }
      ...
  }
}
```

创建子 fiber 的过程会进入 reconcileChildren，用来生成子 fiber。mount 和 update 两种情况处理的区别在于是否追踪副作用，为对应的节点打上 effectTag，之后在 commit 阶段会被执行对应 dom 的增删改。

ChildReconciler 中的对 workInProgress 树的节点打上 effect tag，workInProgress 树如果复用于上一次的 current 树，打上标签后在 commit 阶段操作 dom 时就可以减少很多 dom 操作

```js
//ReactFiberBeginWork.old.js
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes
) {
  if (current === null) {
    //mount时
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes
    );
  } else {
    //update
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes
    );
  }
}

var reconcileChildFibers = ChildReconciler(true);
var mountChildFibers = ChildReconciler(false);
function ChildReconciler(shouldTrackSideEffects) {
  function placeChild(newFiber, lastPlacedIndex, newIndex) {
    newFiber.index = newIndex;

    if (!shouldTrackSideEffects) {
      //是否追踪副作用
      // Noop.
      return lastPlacedIndex;
    }

    var current = newFiber.alternate;

    if (current !== null) {
      var oldIndex = current.index;

      if (oldIndex < lastPlacedIndex) {
        // This is a move.
        newFiber.flags = Placement;
        return lastPlacedIndex;
      } else {
        // This item can stay in place.
        return oldIndex;
      }
    } else {
      // This is an insertion.
      newFiber.flags = Placement;
      return lastPlacedIndex;
    }
  }
}
```

以下是常见的 effectTag

```js
export const Placement = /*             */ 0b0000000000010; // 插入节点
export const Update = /*                */ 0b0000000000100; // 更新fiber
export const Deletion = /*              */ 0b0000000001000; // 删除fiebr
export const Snapshot = /*              */ 0b0000100000000; // 快照
export const Passive = /*               */ 0b0001000000000; // useEffect的副作用
export const Callback = /*              */ 0b0000000100000; // setState的 callback
export const Ref = /*                   */ 0b0000010000000; // ref
```

#### completeWork

completeWork 主要工作是处理 fiber 的 props、创建 dom、创建 effectList

- mount：调用 createInstance 创建 dom，将后代 dom 节点插入刚创建的 dom 中，并处理 props

- update：处理 props，最后会保存在 workInProgress.updateQueue 上

```js
//ReactFiberCompleteWork.old.js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  //根据workInProgress.tag进入不同逻辑，以HostComponent为例
  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case HostRoot:
    //...

    case HostComponent: {
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;

      if (current !== null && workInProgress.stateNode != null) {
        // update时
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance
        );
      } else {
        // mount时
        const currentHostContext = getHostContext();
        // 创建fiber对应的dom节点
        const instance = createInstance(
          type,
          newProps,
          rootContainerInstance,
          currentHostContext,
          workInProgress
        );
        // 将后代dom节点插入刚创建的dom里
        appendAllChildren(instance, workInProgress, false, false);
        // dom节点赋值给fiber.stateNode
        workInProgress.stateNode = instance;

        // 处理props和updateHostComponent类似
        if (
          finalizeInitialChildren(
            instance,
            type,
            newProps,
            rootContainerInstance,
            currentHostContext
          )
        ) {
          markUpdate(workInProgress);
        }
      }
      return null;
    }
  }
}
```

同时向上回溯的过程中遇到带 effectTag 的节点，会将这个节点加入 effectList 中，在 commit 阶段只要遍历 effectList 就可以了。effectList 是一个环状链表，从触发更新的节点向上合并 effectList 直到 rootFiber，这一过程发生在 completeUnitOfWork 函数中

```js
//ReactFiberWorkLoop.old.js
function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  do {
    	//...

      if (
        returnFiber !== null &&
        (returnFiber.flags & Incomplete) === NoFlags
      ) {
        if (returnFiber.firstEffect === null) {
          returnFiber.firstEffect = completedWork.firstEffect;//父节点的effectList头指针指向completedWork的effectList头指针
        }
        if (completedWork.lastEffect !== null) {
          if (returnFiber.lastEffect !== null) {
            //父节点的effectList头尾指针指向completedWork的effectList头指针
            returnFiber.lastEffect.nextEffect = completedWork.firstEffect;
          }
          //父节点头的effectList尾指针指向completedWork的effectList尾指针
          returnFiber.lastEffect = completedWork.lastEffect;
        }

        const flags = completedWork.flags;
        if (flags > PerformedWork) {
          if (returnFiber.lastEffect !== null) {
            //completedWork本身追加到returnFiber的effectList结尾
            returnFiber.lastEffect.nextEffect = completedWork;
          } else {
            //returnFiber的effectList头节点指向completedWork
            returnFiber.firstEffect = completedWork;
          }
          //returnFiber的effectList尾节点指向completedWork
          returnFiber.lastEffect = completedWork;
        }
      }
    } else {

      //...

      if (returnFiber !== null) {
        returnFiber.firstEffect = returnFiber.lastEffect = null;//重制effectList
        returnFiber.flags |= Incomplete;
      }
    }

  } while (completedWork !== null);

	//...
}
```

例如下面的代码及对应的 effectList

```js
function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <h1
        onClick={() => {
          setCount(() => count + 1);
        }}
      >
        <p title={count}>{count}</p>
      </h1>
    </>
  );
}
```

![](https://gitee.com/xiaochen1024/assets/raw/master/assets/20210529105807.png)

参考

1. [react 源码解析 8.render 阶段](https://xiaochen1024.com/courseware/60b1b2f6cf10a4003b634718/60b1b348cf10a4003b634720)
