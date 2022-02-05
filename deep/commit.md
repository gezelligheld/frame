更新 fiber 的过程称为调和（Reconciler），其中又分为 render 阶段和 commit 阶段。在 render 阶段的末尾会调用 commitRoot(root)后进入 commit 阶段，commit 阶段的主要工作是进行节点的增删改最终渲染到页面上

主体逻辑如下，分别执行三个方法 commitBeforeMutationEffects、commitMutationEffects、commitLayoutEffects 执行对应的 dom 操作和生命周期

```js
function commitRootImpl(root, renderPriorityLevel) {
  //...
  do {
    //...
    commitBeforeMutationEffects();
  } while (nextEffect !== null);

  do {
    //...
    commitMutationEffects(root, renderPriorityLevel); //commitMutationEffects
  } while (nextEffect !== null);

  root.current = finishedWork; //切换current Fiber树

  do {
    //...
    commitLayoutEffects(root, lanes); //commitLayoutEffects
  } while (nextEffect !== null);
  //...
}
```

#### Before Mutation

执行了 commitBeforeMutationEffects 函数，同步执行了 getSnapshotBeforeUpdate 生命周期；执行对应的 useEffect 回调和销毁函数

```js
function flushPassiveEffectsImpl() {
  if (rootWithPendingPassiveEffects === null) {
    //在mutation后变成了root
    return false;
  }
  const unmountEffects = pendingPassiveHookEffectsUnmount;
  pendingPassiveHookEffectsUnmount = []; //useEffect的回调函数
  for (let i = 0; i < unmountEffects.length; i += 2) {
    const effect = ((unmountEffects[i]: any): HookEffect);
    //...
    const destroy = effect.destroy;
    destroy();
  }

  const mountEffects = pendingPassiveHookEffectsMount; //useEffect的销毁函数
  pendingPassiveHookEffectsMount = [];
  for (let i = 0; i < mountEffects.length; i += 2) {
    const effect = ((unmountEffects[i]: any): HookEffect);
    //...
    const create = effect.create;
    effect.destroy = create();
  }
}
```

#### Mutation

执行 commitMutationEffects 函数，解绑 ref；根据 effectTag 执行对应的 dom 操作；useLayoutEffect 销毁函数在 UpdateTag 时执行

```js
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  //遍历effectList
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag;
    // 调用commitDetachRef解绑ref
    if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }

    // 根据effectTag执行对应的dom操作
    const primaryEffectTag =
      effectTag & (Placement | Update | Deletion | Hydrating);
    switch (primaryEffectTag) {
      // 插入dom
      case Placement: {
        commitPlacement(nextEffect);
        nextEffect.effectTag &= ~Placement;
        break;
      }
      // 插入更新dom
      case PlacementAndUpdate: {
        // 插入
        commitPlacement(nextEffect);
        nextEffect.effectTag &= ~Placement;
        // 更新
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      //...
      // 更新dom
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 删除dom
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```

插入节点的函数 commitPlacement，根据 isContainer 来判断是插入到兄弟节点前还是 append 到父节点后

```js
function commitPlacement(finishedWork: Fiber): void {
  //...
  const parentFiber = getHostParentFiber(finishedWork); //找到最近的parent

  let parent;
  let isContainer;
  const parentStateNode = parentFiber.stateNode;
  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentStateNode;
      isContainer = false;
      break;
    //...
  }
  const before = getHostSibling(finishedWork); //找兄弟节点
  if (isContainer) {
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
  } else {
    insertOrAppendPlacementNode(finishedWork, before, parent);
  }
}
```

更新节点的函数 commitWork，会执行 useLayoutEffect 的销毁函数，最终 commitUpdate 调用 updateDOMProperties 更新 dom

```js
function commitWork(current: Fiber | null, finishedWork: Fiber): void {
  if (!supportsMutation) {
    switch (finishedWork.tag) {
      //...
      case SimpleMemoComponent: {
        commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork);
      }
      //...
    }
  }

  switch (finishedWork.tag) {
    //...
    case HostComponent:
      {
        //...
        commitUpdate(
          instance,
          updatePayload,
          type,
          oldProps,
          newProps,
          finishedWork
        );
      }
      return;
  }
}

function updateDOMProperties(
  domElement: Element,
  updatePayload: Array<any>,
  wasCustomComponentTag: boolean,
  isCustomComponentTag: boolean
): void {
  // TODO: Handle wasCustomComponentTag
  for (let i = 0; i < updatePayload.length; i += 2) {
    const propKey = updatePayload[i];
    const propValue = updatePayload[i + 1];
    if (propKey === STYLE) {
      setValueForStyles(domElement, propValue);
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
      setInnerHTML(domElement, propValue);
    } else if (propKey === CHILDREN) {
      setTextContent(domElement, propValue);
    } else {
      setValueForProperty(domElement, propKey, propValue, isCustomComponentTag);
    }
  }
}
```

删除节点的函数 commitDeletion 函数，类组件会执行 componentWillUnmount，函数组件会删除 ref、并执行 useEffect 的销毁函数

```js
function commitDeletion(
  finishedRoot: FiberRoot,
  current: Fiber,
  renderPriorityLevel: ReactPriorityLevel
): void {
  if (supportsMutation) {
    // Recursively delete all host nodes from the parent.
    // Detach refs and call componentWillUnmount() on the whole subtree.
    unmountHostComponents(finishedRoot, current, renderPriorityLevel);
  } else {
    // Detach refs and call componentWillUnmount() on the whole subtree.
    commitNestedUnmounts(finishedRoot, current, renderPriorityLevel);
  }
  const alternate = current.alternate;
  detachFiberMutation(current);
  if (alternate !== null) {
    detachFiberMutation(alternate);
  }
}
```

#### Layout

执行 commitLayoutEffects 函数

```js
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag;

    // 调用commitLayoutEffectOnFiber执行生命周期和hook
    if (effectTag & (Update | Callback)) {
      const current = nextEffect.alternate;
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes);
    }

    // ref赋值
    if (effectTag & Ref) {
      commitAttachRef(nextEffect);
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```

重新赋值 ref 或触发 ref 回调

```js
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;

    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent:
        instanceToUse = getPublicInstance(instance);
        break;
      default:
        instanceToUse = instance;
    }

    if (typeof ref === 'function') {
      // 执行ref回调
      ref(instanceToUse);
    } else {
      // 如果是值的类型则赋值给ref.current
      ref.current = instanceToUse;
    }
  }
}
```

函数组件会执行 useLayoutEffect 的回调、异步调度 useEffect 的回调和销毁函数，类组件会执行 componentDidMount 或者 componentDidUpdate，以及 setState 的 callback 回调

```js
function commitLifeCycles(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedLanes: Lanes
): void {
  switch (finishedWork.tag) {
    case SimpleMemoComponent: {
      // 此函数会调用useLayoutEffect的回调
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
      // 向pendingPassiveHookEffectsUnmount和pendingPassiveHookEffectsMount中push effect并且调度它们
      schedulePassiveEffects(finishedWork);
    }
    case ClassComponent: {
      //条件判断...
      instance.componentDidMount();
      //条件判断...
      instance.componentDidUpdate(
        //update 在layout期间同步执行
        prevProps,
        prevState,
        instance.__reactInternalSnapshotBeforeUpdate
      );
    }

    case HostRoot: {
      commitUpdateQueue(finishedWork, updateQueue, instance); //render第三个参数
    }
  }
}
function schedulePassiveEffects(finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      const { next, tag } = effect;
      if (
        (tag & HookPassive) !== NoHookEffect &&
        (tag & HookHasEffect) !== NoHookEffect
      ) {
        //push useEffect的销毁函数并且加入调度
        enqueuePendingPassiveHookEffectUnmount(finishedWork, effect);
        //push useEffect的回调函数并且加入调度
        enqueuePendingPassiveHookEffectMount(finishedWork, effect);
      }
      effect = next;
    } while (effect !== firstEffect);
  }
}
```

> useLayoutEffect 是在 commit 阶段同步执行，useEffect 会在 commit 阶段异步调度

参考

1. [react 源码解析 10.commit 阶段](https://xiaochen1024.com/courseware/60b1b2f6cf10a4003b634718/60b1b360cf10a4003b634722)
