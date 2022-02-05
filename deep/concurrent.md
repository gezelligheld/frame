concurrent 模式是一组新功能的总称，包括 Fiber、Scheduler、Lane，目的是让应用保持对 cpu 和 io 的快速响应

#### Fiber

Fiber 在 concurrent 下的意义在于，当当前任务比较耗时的时候，可以主动中断从而执行更高优先级的任务

#### Scheduler

Scheduler 的意义在于，对每一帧进行时间切片用来执行任务，从而使页面看起来不卡顿。当任务执行超过一帧的时间时，就会暂停任务的执行，让浏览器执行重排和重绘，然后在下一帧空闲的时间继续执行

#### lane

Lane 用二进制位表示任务的优先级，方便优先级的计算，不同优先级占用不同位置的‘赛道’，而且存在批的概念，优先级越低，‘赛道’越多

#### batchedUpdates

合并更新。在之前的版本中如果脱离 react 的上下文则更新不会被合并，如下

```js
onClick() {
 setTimeout(() => {
    this.setState({ count: this.state.count + 1 });
    this.setState({ count: this.state.count + 1 });
  });
}
```

原因在于，合并更新时当 executionContext 不处于 BatchedContext 下时就会同步执行，不作合并处理

```js
export function batchedUpdates<A, R>(fn: (A) => R, a: A): R {
  const prevExecutionContext = executionContext;
  // 按位或然后赋值
  executionContext |= BatchedContext;
  try {
    return fn(a);
  } finally {
    executionContext = prevExecutionContext;
    if (executionContext === NoContext) {
      resetRenderTimer();
      //executionContext为NoContext就同步执行SyncCallbackQueue中的任务
      flushSyncCallbackQueue();
    }
  }
}
```

在 concurrent 下上述的情况也会被合并更新，如果多次 setState，会比较这几次 setState 回调的优先级，如果优先级一致，则先 return 掉，不会进行后面的 render 阶段

```js
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode; //之前已经调用过的setState的回调
  //...
  if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority;
    //新的setState的回调和之前setState的回调优先级相等 则进入batchedUpdate的逻辑
    if (existingCallbackPriority === newCallbackPriority) {
      return;
    }
    cancelCallback(existingCallbackNode);
  }
  //调度render阶段的起点
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root)
  );
  //...
}
```

#### Suspense

Suspense 可以在请求数据的时候显示 pending 状态，请求成功后展示数据，原因是因为 Suspense 中组件的优先级很低，而离屏的 fallback 组件优先级高，当 Suspense 中组件 resolve 之后就会重新调度一次 render 阶段去渲染已经准备好的数据
