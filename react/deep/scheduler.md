浏览器的渲染线程和 js 线程是互斥的，当执行 js 的时候，渲染线程会挂起，时间长了就会造成页面卡顿的感觉。16 版本之前的 react，如果状态发生更新，会从根节点开始进行 diff，递归遍历所有的虚拟 DOM 找到不同，然后更新 DOM，这样同步更新的方式可能会造成卡顿

对于函数来说，这没什么问题，因为我们只想要函数的运行结果，但对于 UI 来说还需要考虑以下问题:

- 并不是所有的 state 更新都需要立即显示出来，比如屏幕之外的部分的更新
- 并不是所有的更新优先级都是一样的
- 对于某些高优先级的操作，应该是可以打断低优先级的操作执行的

16 版本后采用了新的调度方式，react 实现了一套自己的调度方式，更新任务可以分成很多小部分，甚至可以中断，以一个 fiber 作为最小的工作单元执行完毕后，都会去查看是否还继续拥有主线程时间片，如果有继续执行下一个 fiber 的更新任务，没有则进行其他高优的任务，等主线程进入空闲时间后再继续

Scheduler 作为 react 中一个独立的包，承担着任务调度的职责，只需要将任务优先级和任务提交给它，就可以帮助管理任务

#### Concurrent 模式独有

在 Legacy 模式下，没有额外的调度处理和优先级之分，会按顺序处理多个更新任务，直接进入调和流程，这样容易造成浏览器卡顿；Concurrent 模式下，会优先处理优先级较高的任务，如果在执行调和 render 阶段时出现了更高优先级的任务就会打断当前任务，而去执行该任务

#### 时间分片

浏览器每次执行一次事件循环（一帧）都会做如下事情：处理事件，执行 js ，调用 requestAnimation ，布局 Layout ，绘制 Paint ，在一帧执行后，如果没有其他事件，那么浏览器会进入空闲时间，那么有的一些不是特别紧急 React 更新，就可以执行了

时间分片规定的是单个任务在这一帧内最大的执行时间，任务一旦执行时间超过时间片，则会被打断，有节制地执行任务。这样可以保证页面不会因为任务连续执行的时间过长而产生卡顿

浏览器提供的 requestIdleCallback，当进入空闲时间后，触发其回调

```js
// timeout 超时时间，如果长时间没有空闲时间回调不会执行，超过超时时间会强制执行
requestIdleCallback(callback, { timeout });
```

为了防止 requestIdleCallback 中的任务由于浏览器没有空闲时间而卡死，所以设置了 5 个优先级

- Immediate -1 需要立刻执行。
- UserBlocking 250ms 超时时间 250ms，一般指的是用户交互。
- Normal 5000ms 超时时间 5s，不需要直观立即变化的任务，比如网络请求。
- Low 10000ms 超时时间 10s，肯定要执行的任务，但是可以放在最后处理。
- Idle 一些没有必要的任务，可能不会执行

react 通过类似 requestIdleCallback 去向浏览器做一帧一帧请求，等到浏览器有空余时间，去执行异步更新任务，来保证页面的流畅度

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cdece5756244975beb3ca5352af4eb8~tplv-k3u1fbpfcp-watermark.awebp)

为了让视图流畅地运行，可以按照人类能感知到最低限度每秒 60 帧的频率划分时间片，这样每个时间片就是 16ms 。也就是这 16 毫秒要完成如上 js 执行，浏览器绘制等操作，由于 requestIdleCallback 只有谷歌浏览器支持，react Scheduler 内部借助 MessageChannel 实现了在浏览器绘制之前指定一个时间片，如果 react 在指定时间内没处理完，Scheduler 就会强制交出执行权给浏览器

```js
let scheduledHostCallback = null;
/* 建立一个消息通道 */
var channel = new MessageChannel();
/* 建立一个port发送消息 */
var port = channel.port2;

channel.port1.onmessage = function () {
  /* 执行任务 */
  scheduledHostCallback();
  /* 执行完毕，清空任务 */
  scheduledHostCallback = null;
};
/* 向浏览器请求执行更新任务 */
requestHostCallback = function (callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    port.postMessage(null);
  }
};
```

在一次更新任务中，react 调用 requestHostCallback，port2 项 port1 发起消息通知，port1 的 onmessage 触发，执行异步更新任务

#### 调度优先级

- runWithPriority

以一个优先级执行 callback，如果是同步的任务，优先级就是 ImmediateSchedulerPriority

```js
function unstable_runWithPriority(priorityLevel, eventHandler) {
  switch (
    priorityLevel //5种优先级
  ) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  var previousPriorityLevel = currentPriorityLevel; //保存当前的优先级
  currentPriorityLevel = priorityLevel; //priorityLevel赋值给currentPriorityLevel

  try {
    return eventHandler(); //回调函数
  } finally {
    currentPriorityLevel = previousPriorityLevel; //还原之前的优先级
  }
}
```

- scheduleCallback

以一个优先级注册 callback，在适当的时机执行，因为涉及过期时间的计算，所以 scheduleCallback 比 runWithPriority 的粒度更细。在这个函数中，优先级意味着过期时间，优先级越高 priorityLevel 就越小，过期时间离当前时间就越近

未过期任务 task 存放在 timerQueue 中，过期任务存放在 taskQueue 中，timerQueue 中的任务如果过期了会加入 taskQueue 中

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;//开始时间
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority://优先级越高timeout越小
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;//-1
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;//250
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  var expirationTime = startTime + timeout;//优先级越高 过期时间越小

  var newTask = {//新建task
    id: taskIdCounter++,
    callback//回调函数
    priorityLevel,
    startTime,//开始时间
    expirationTime,//过期时间
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }

  if (startTime > currentTime) {//没有过期
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);//加入timerQueue
    //taskQueue中还没有过期任务，timerQueue中离过期时间最近的task正好是newTask
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      if (isHostTimeoutScheduled) {
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      //定时器，到了过期时间就加入taskQueue中
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);//加入taskQueue
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);//执行过期的任务
    }
  }

  return newTask;
}
```

调度的过程用到了小顶堆，可以在 O(1)的复杂度找到优先级最高的 task。小顶堆的特点是每一个父节点都比它的两个子节点小，第一个节点是最小的

- 每次 peek 时，获取小顶堆的第一个任务，即优先级最高的任务

- 每次 push 时，将 push 的这个节点与父节点比较，如果父节点较大则交换位置，否则退出循环

```js
function push(heap: Heap, node: Node): void {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);
}
function siftUp(heap, node, i) {
  let index = i;
  while (index > 0) {
    const parentIndex = (index - 1) >>> 1;
    const parent = heap[parentIndex];
    if (compare(parent, node) > 0) {
      // The parent is larger. Swap positions.
      heap[parentIndex] = node;
      heap[index] = parent;
      index = parentIndex;
    } else {
      // The parent is smaller. Exit.
      return;
    }
  }
}
// 优先比较任务优先级，其次比较插入顺序
function compare(a, b) {
  // Compare sort index first, then task id.
  const diff = a.sortIndex - b.sortIndex;
  return diff !== 0 ? diff : a.id - b.id;
}
```

- 每次 pop 时，pop 的实际是优先级最高的任务，然后再重新排序

```js
function pop(heap: Heap): Node | null {
  if (heap.length === 0) {
    return null;
  }
  const first = heap[0];
  const last = heap.pop();
  if (last !== first) {
    heap[0] = last;
    siftDown(heap, last, 0);
  }
  return first;
}
function siftDown(heap, node, i) {
  let index = i;
  const length = heap.length;
  const halfLength = length >>> 1;
  while (index < halfLength) {
    const leftIndex = (index + 1) * 2 - 1;
    const left = heap[leftIndex];
    const rightIndex = leftIndex + 1;
    const right = heap[rightIndex];

    // If the left or right node is smaller, swap with the smaller of those.
    if (compare(left, node) < 0) {
      if (rightIndex < length && compare(right, left) < 0) {
        heap[index] = right;
        heap[rightIndex] = node;
        index = rightIndex;
      } else {
        heap[index] = left;
        heap[leftIndex] = node;
        index = leftIndex;
      }
    } else if (rightIndex < length && compare(right, node) < 0) {
      heap[index] = right;
      heap[rightIndex] = node;
      index = rightIndex;
    } else {
      // Neither child is smaller. Exit.
      return;
    }
  }
}
```

#### 调度流程

ReactDOM.render、setState、useState 都会执行 scheduleUpdateOnFiber，可以将其看作一个更新入口。初始化时会直接执行 performSyncWorkOnRoot 进入调和流程，使用 setState/useState 时会执行 ensureRootIsScheduled 进入调度流程

```js
export function scheduleUpdateOnFiber(fiber, lane, eventTime) {
  if (lane === SyncLane) {
    if (
      (executionContext & LegacyUnbatchedContext) !== NoContext && // unbatch 情况，比如初始化
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      /* 开始同步更新，进入到 调和 流程 */
      performSyncWorkOnRoot(root);
    } else {
      /* 进入调度，把任务放入调度中 */
      ensureRootIsScheduled(root, eventTime);
      if (executionContext === NoContext) {
        /* 当前的执行任务类型为 NoContext ，说明当前任务是非可控的，那么会调用 flushSyncCallbackQueue 方法。 */
        flushSyncCallbackQueue();
      }
    }
  }
}
```

legacy 模式下，宏队列和微队列中的更新任务不会处理为批量更新，原因在于，对于 react 系统中的事件都会标识为相应的上下文，也就是说 executionContext 不为 NoContext，但宏队列和微队列中的更新任务无法控制其执行时机，就会直接触发 flushSyncCallbackQueue 执行更新队列中的任务，直接进入调和流程

进入调度流程后，如果最新的调度更新优先级和当前 root 的优先级一样，则退出调度流程，否则将当前更新任务放入执行队列，然后依次通过 MessageChannel 请求空闲帧从而执行队列中的任务，

```js
function ensureRootIsScheduled(root, currentTime) {
  /* 计算一下执行更新的优先级 */
  var newCallbackPriority = returnNextLanesPriority();
  /* 当前 root 上存在的更新优先级 */
  const existingCallbackPriority = root.callbackPriority;
  /* 如果两者相等，那么说明是在一次更新中，那么将退出 */
  if (existingCallbackPriority === newCallbackPriority) {
    return;
  }
  if (newCallbackPriority === SyncLanePriority) {
    /* 在正常情况下，会直接进入到调度任务中。 */
    newCallbackNode = scheduleSyncCallback(
      performSyncWorkOnRoot.bind(null, root)
    );
  } else {
    /* 这里先忽略 */
  }
  /* 给当前 root 的更新优先级，绑定到最新的优先级  */
  root.callbackPriority = newCallbackPriority;
}

function scheduleSyncCallback(callback) {
  if (syncQueue === null) {
    /* 如果队列为空 */
    syncQueue = [callback];
    /* 放入调度任务 */
    immediateQueueCallbackNode = Scheduler_scheduleCallback(
      Scheduler_ImmediatePriority,
      flushSyncCallbackQueueImpl
    );
  } else {
    /* 如果任务队列不为空，那么将任务放入队列中。 */
    syncQueue.push(callback);
  }
}
```

#### 任务的暂停和继续

时间分片的计算如下，默认为 5ms

```js
function forceFrameRate(fps) {
  //计算时间片
  if (fps > 0) {
    yieldInterval = Math.floor(1000 / fps);
  } else {
    yieldInterval = 5; //时间片默认5ms
  }
}
```

当前时间大于任务开始的时间+yieldInterval 时，就会打断任务的进行

```js
//deadline = currentTime + yieldInterval，deadline是在performWorkUntilDeadline函数中计算出来的
if (currentTime >= deadline) {
  //...
  return true;
}
```

当更高优先级的渲染任务到来时 React 在此时会执行以下操作

1. 记录中断位置并中断当前渲染任务，及时响应高优先级任务
2. 回滚到之前的状态，React 使用 alternate 指针回滚到之前的状态，恢复之前更新的 Fiber 树
3. 当高优先级任务执行完毕后，React 会重新调度被中断的渲染任务，并从中断位置继续渲染

1. [react concurrent](https://zhuanlan.zhihu.com/p/60307571?utm_source=tuicool)
2. [react 源码解析 15.scheduler&Lane](https://xiaochen1024.com/courseware/60b1b2f6cf10a4003b634718/60b1b556cf10a4003b634727)
3. [某大厂一面: 讲讲 Fiber 架构以及它是如何实现增量渲染的🥱🥱🥱](https://juejin.cn/post/7258881840823844920)