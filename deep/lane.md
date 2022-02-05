Lane 和 Scheduler 是两套优先级机制，相比来说 Lane 的优先级粒度更细

每个优先级都是个 31 位二进制数字，1 表示该位置可以用，0 代表这个位置不能用，从第一个优先级 NoLanes 到 OffscreenLane 优先级是降低的，优先级越低 1 的个数也就越多

```js
export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;
export const SyncBatchedLane: Lane = /*                 */ 0b0000000000000000000000000000010;

export const InputDiscreteHydrationLane: Lane = /*      */ 0b0000000000000000000000000000100;
const InputDiscreteLanes: Lanes = /*                    */ 0b0000000000000000000000000011000;

const InputContinuousHydrationLane: Lane = /*           */ 0b0000000000000000000000000100000;
const InputContinuousLanes: Lanes = /*                  */ 0b0000000000000000000000011000000;

export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000100000000;
export const DefaultLanes: Lanes = /*                   */ 0b0000000000000000000111000000000;

const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000001000000000000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111110000000000000;

const RetryLanes: Lanes = /*                            */ 0b0000011110000000000000000000000;

export const SomeRetryLane: Lanes = /*                  */ 0b0000010000000000000000000000000;

export const SelectiveHydrationLane: Lane = /*          */ 0b0000100000000000000000000000000;

const NonIdleLanes = /*                                 */ 0b0000111111111111111111111111111;

export const IdleHydrationLane: Lane = /*               */ 0b0001000000000000000000000000000;
const IdleLanes: Lanes = /*                             */ 0b0110000000000000000000000000000;

export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

#### Lane 模型中 task 是怎么获取优先级的

任务获取优先级的方式是从高优先级的 lanes 开始的，这个过程发生在 findUpdateLane 函数中，如果高优先级没有可用的 lane 了就下降到优先级低的 lanes 中寻找，其中 pickArbitraryLane 会调用 getHighestPriorityLane 获取一批 lanes 中优先级最高的那一位

```js
export function findUpdateLane(
  lanePriority: LanePriority,
  wipLanes: Lanes
): Lane {
  switch (lanePriority) {
    //...
    case DefaultLanePriority: {
      let lane = pickArbitraryLane(DefaultLanes & ~wipLanes); //找到下一个优先级最高的lane
      if (lane === NoLane) {
        //上一个level的lane都占满了下降到TransitionLanes继续寻找可用的赛道
        lane = pickArbitraryLane(TransitionLanes & ~wipLanes);
        if (lane === NoLane) {
          //TransitionLanes也满了
          lane = pickArbitraryLane(DefaultLanes); //从DefaultLanes开始找
        }
      }
      return lane;
    }
  }
}
```

#### Lane 模型中高优先级是怎么插队的

在 Lane 模型中如果一个低优先级的任务执行，并且还在调度的时候触发了一个高优先级的任务，则高优先级的任务打断低优先级任务，此时应该先取消低优先级的任务，因为此时低优先级的任务可能已经进行了一段时间，Fiber 树已经构建了一部分，所以需要将 Fiber 树还原，这个过程发生在函数 prepareFreshStack 中，在这个函数中会初始化已经构建的 Fiber 树

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
    //两个回调优先级不一致，则被高优先级任务打断，先取消当前低优先级的任务
    cancelCallback(existingCallbackNode);
  }
  //调度render阶段的起点
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root)
  );
  //...
}
function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  //...
  //workInProgressRoot等变量重新赋值和初始化
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  workInProgressRootRenderLanes =
    subtreeRenderLanes =
    workInProgressRootIncludedLanes =
      lanes;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;
  //...
}
```

#### Lane 模型中怎么解决饥饿问题

在调度优先级的过程中，会遍历未执行的任务包含的 lane（pendingLanes），如果没过期时间就计算一个过期时间，如果过期了就加入 root.expiredLanes 中，下次优先执行。也就是说，随着时间的推移，未被执行的低优先级任务，最终会变成高优先级任务执行

```js
export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number
): void {
  const pendingLanes = root.pendingLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  const expirationTimes = root.expirationTimes;

  let lanes = pendingLanes;
  while (lanes > 0) {
    //遍历lanes
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;

    const expirationTime = expirationTimes[index];
    if (expirationTime === NoTimestamp) {
      if (
        (lane & suspendedLanes) === NoLanes ||
        (lane & pingedLanes) !== NoLanes
      ) {
        expirationTimes[index] = computeExpirationTime(lane, currentTime); //计算过期时间
      }
    } else if (expirationTime <= currentTime) {
      //过期了
      root.expiredLanes |= lane; //在expiredLanes加入当前遍历到的lane
    }

    lanes &= ~lane;
  }
}

export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  //...
  if (expiredLanes !== NoLanes) {
    nextLanes = expiredLanes;
    nextLanePriority = return_highestLanePriority = SyncLanePriority; //优先返回过期的lane
  } else {
    //...
  }
  return nextLanes;
}
```
