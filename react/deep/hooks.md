#### 设计初衷

react 组件复用，经历了 mixins -> 高阶组件 HOC -> 自定义 hook 的过程，类组件状态难以复用，借助高阶组件嵌套层级过多，逻辑复杂，难以维护

16.8 版本之前，组件分为两种，含有状态的类组件和无状态的函数组件

如果想要去 class 改造函数组件的话，函数组件需要拥有和类组件一样的能力，即

- class 组件可以存储实例状态，挂到 this 上，而函数组件每次执行后，状态都会被初始化
- class 组件拥有 setState 方法改变状态，而函数组件没有

而 hooks 的出现，使函数组件也拥有了维持状态、改变状态的能力，自定义 hook 也使得逻辑复用更加容易

#### hook 和 fiber

类组件的状态如 state、props、context 本质上是存在类组件对应的 fiber 上，hooks 既然赋予了函数组件如上的能力，也离不开函数组件对应的 fiber

hooks 对象主要有三种处理策略

1. ContextOnlyDispatcher：防止在函数组件外调用 hooks，否则抛出异常
2. HooksDispatcherOnMount：函数组件初始化时，初始化 hooks，建立起其 fiber 和 hooks 的关系
3. HooksDispatcherOnUpdate：函数组件更新时，需要 hooks 去获取或更新状态

```js
const HooksDispatcherOnMount = { /* 函数组件初始化用的 hooks */
    useState: mountState,
    useEffect: mountEffect,
    ...
}
const  HooksDispatcherOnUpdate ={/* 函数组件更新用的 hooks */
   useState:updateState,
   useEffect: updateEffect,
   ...
}
const ContextOnlyDispatcher = {  /* 当hooks不是函数内部调用的时候，调用这个hooks对象下的hooks，所以报错。 */
   useEffect: throwInvalidHookError,
   useState: throwInvalidHookError,
   ...
}
```

函数组件的多个 hook 会形成 hook 链表，保存在 Fiber 的 memoizedState 的上，hook 数据结构如下

```js
const hook: Hook = {
  // 对于const [state, updateState] = useState(initialState)，memoizedState = state
  // 对于const [state, dispatch] = useReducer(reducer, {})，memoizedState = state
  // 对于useEffect(callback, [dep])，memoizedState是effect链表，每个节点是{create:callback, dep:dep,...}这样的形式
  // 对于useRef(0)，memoizedState = {current:0}
  // 对于useMemo(callback, [dep])，memoizedState = [callback(), dep]
  // 对于useCallback(callback, [dep])，memoizedState = [callback, dep]
  memoizedState: null, //对于不同hook，有不同的值
  baseState: null, //初始state
  baseQueue: null, //初始queue队列
  queue: null, //需要更新的update
  next: null, //下一个hook
};
```

##### 函数组件触发

在 fiber 调和的过程中，遇到函数组件类型的 fiber，会调用 updateFunctionComponent 中的 renderWithHooks 更新 fiber

```js
let currentlyRenderingFiber;
function renderWithHooks(current, workInProgress, Component, props) {
  currentlyRenderingFiber = workInProgress;
  workInProgress.memoizedState =
    null; /* 每一次执行函数组件之前，先清空状态 （用于存放hooks列表）*/
  workInProgress.updateQueue = null; /* 清空状态（用于存放effect list） */
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate; /* 判断是初始化组件还是更新组件 */
  let children = Component(
    props,
    secondArg
  ); /* 执行我们真正函数组件，所有的hooks将依次执行。 */
  ReactCurrentDispatcher.current =
    ContextOnlyDispatcher; /* 将hooks变成第一种，防止hooks在函数组件外部调用，调用直接报错。 */
}
```

其中

- workInProgress 是正在调和的当前函数组件对应的 fiber 树
- 对于类组件，memoizedState 存储的是 state 信息，对于函数组件 memoizedState 存储的是 hooks 信息
- 对于函数组件，updateQueue 用于存放 useEffect/useLayoutEffect 产生的 effect list，用于在最终的 commit 阶段执行副作用
- 引用的 React hooks 都是从 ReactCurrentDispatcher.current 中的， React 就是通过赋予 current 不同的 hooks 对象达到监控 hooks 是否在函数组件内部调用
- 真正的函数组件 Component 也在此时执行，里面的每个 hooks 也依次执行

#### useState & useReducer

useState 和 useReducer 让函数组件拥有了状态和改变状态的能力

初始化时 useState 调用 mountState，useReducer 调用 mountReducer，两者唯一的区别在于 mountState 的 lastRenderedReducer 给了初始值，可以说 useState 就是有默认 reducer 参数的 useReducer

```ts
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();//创建当前hook
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;//hook.memoizedState赋值
  const queue = (hook.queue = {//赋值hook.queue
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,//和mountReducer的区别
    lastRenderedState: (initialState: any),
  });
  //创建dispatch函数
  const dispatch: Dispatch<BasicStateAction<S>> = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ) as any));
  return [hook.memoizedState, dispatch];//返回memoizedState和dispatch
}

function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = mountWorkInProgressHook();//创建当前hook
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = ((initialArg: any): S);
  }
  hook.memoizedState = hook.baseState = initialState;//hook.memoizedState赋值
  const queue = (hook.queue = {//创建queue
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(//创建dispatch函数
    null,
    currentlyRenderingFiber,
    queue,
  ) as any));
  return [hook.memoizedState, dispatch];//返回memoizedState和dispatch
}

function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}
```

更新时都调用 updateReducer，根据 hook 中的 update 计算新的 state

```ts
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook(); //获取hook
  const queue = hook.queue;
  queue.lastRenderedReducer = reducer;
  //...
  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```

每次使用 useState 和 useReducer 改变 state，其实调用的是 dispatchAction

- 首先创建一个 update，将其放入待更新的队列 pending 中
- 当前 fiber 没有更新任务时，将这一次的 state 和上一次的 state 进行浅比较，如果相同退出更新，否则发起更新调度任务

```js
function dispatchAction(fiber, queue, action) {
  /* 创建一个 update 对象 */
  var update = {
    eventTime: eventTime,
    lane: lane,
    suspenseConfig: suspenseConfig,
    action: action,
    eagerReducer: null,
    eagerState: null,
    next: null,
  };
  const pending = queue.pending;
  if (pending === null) {
    /* 第一个待更新任务 */
    update.next = update;
  } else {
    /* 已经有带更新任务 */
    update.next = pending.next;
    pending.next = update;
  }
  if (fiber === currentlyRenderingFiber) {
    /* 说明当前fiber正在发生调和渲染更新，那么不需要更新 */
  } else {
    if (
      fiber.expirationTime === NoWork &&
      (alternate === null || alternate.expirationTime === NoWork)
    ) {
      const lastRenderedReducer = queue.lastRenderedReducer;
      const currentState = queue.lastRenderedState; /* 上一次的state */
      const eagerState = lastRenderedReducer(
        currentState,
        action
      ); /* 这一次新的state */
      if (is(eagerState, currentState)) {
        /* 如果每一个都改变相同的state，那么组件不更新 */
        return;
      }
    }
    scheduleUpdateOnFiber(fiber, expirationTime); /* 发起调度更新 */
  }
}
```

如下面的代码

```js
export default function Index() {
  const [number, setNumber] = useState(0);
  const handleClick = () => {
    setNumber((num) => num + 1); // num = 1
    setNumber((num) => num + 2); // num = 3
    setNumber((num) => num + 3); // num = 6
  };
  return (
    <div>
      <button onClick={() => handleClick()}>点击 {number} </button>
    </div>
  );
}
```

点击一次按钮，触发了三次 dispatchAction，会存到 pending 队列中，被合并成一次更新；当再次执行 setNumber 时，会触发更新 hooks 逻辑，调用 updateReducer，会将待更新队列拿出来，合并成最新的值给到 memoizedState，这样再次调用 setNumber 时内部可以拿到最新值

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/750ee5e50ff8494791f52bd095b305ca~tplv-k3u1fbpfcp-watermark.awebp)

#### useEffect & useLayoutEffect

fiber 更新的 render 阶段是没有任何副作用的，想要操作的类型被打上了不同的 effectTag，在 commit 阶段统一处理副作用

对于类组件，有 componentDidMount/componentDidUpdate 固定的生命周期钩子，用于执行初始化/更新的副作用逻辑，但是对于函数组件，可能存在多个 useEffect/useLayoutEffect ，hooks 把这些 effect，独立形成链表结构，在 commit 阶段统一处理和执行

初始化时调用 mountEffect，将 memoizedState 赋值为 effect 链表

```js
function mountEffect(create, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.effectTag |= UpdateEffect | PassiveEffect;
  // 通过 pushEffect 创建一个 effect，并保存到当前 hooks 的 memoizedState 属性
  // 如果存在多个 effect 或者 layoutEffect 会形成一个副作用链表，绑定在函数组件 fiber 的 updateQueue 上
  hook.memoizedState = pushEffect(
    HookHasEffect | hookEffectTag,
    create, // useEffect 第一次参数，就是副作用函数
    undefined,
    nextDeps // useEffect 第二次参数，deps
  );
}
```

更新时浅比较依赖，如果依赖性变了 pushEffect 第一个参数传 HookHasEffect | hookFlags，HookHasEffect 表示 useEffect 依赖项改变了，需要在 commit 阶段重新执行

```js
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        //比较deps
        //即使依赖相等也要将effect加入链表，以保证顺序一致
        pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    //参数传HookHasEffect | hookFlags，包含hookFlags的useEffect会在commit阶段执行这个effect
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps
  );
}
```

useEffect 最终都在 commit 阶段 mutation 之后依次执行上次 render 的销毁函数和本次 render 的回调函数，而 useLayoutEffect 在 commit 阶段 mutation 时执行上次 render 的销毁函数、mutation 之后执行本次 render 的回调函数

```js
const unmountEffects = pendingPassiveHookEffectsUnmount;
pendingPassiveHookEffectsUnmount = [];
for (let i = 0; i < unmountEffects.length; i += 2) {
  const effect = ((unmountEffects[i]: any): HookEffect);
  const fiber = ((unmountEffects[i + 1]: any): Fiber);
  const destroy = effect.destroy;
  effect.destroy = undefined;

  if (typeof destroy === 'function') {
    try {
      destroy(); //销毁函数执行
    } catch (error) {
      captureCommitPhaseError(fiber, error);
    }
  }
}

const mountEffects = pendingPassiveHookEffectsMount;
pendingPassiveHookEffectsMount = [];
for (let i = 0; i < mountEffects.length; i += 2) {
  const effect = ((mountEffects[i]: any): HookEffect);
  const fiber = ((mountEffects[i + 1]: any): Fiber);

  try {
    const create = effect.create; //本次render的创建函数
    effect.destroy = create();
  } catch (error) {
    captureCommitPhaseError(fiber, error);
  }
}
```

#### useRef

useRef 创建并维护一个 ref 对象，用于获取 dom、保存一些状态等，一些不希望引起页面重新渲染的值可以存到 ref 中；另外全局变量也可以用 ref 代替，ref 会随着组件的销毁而销毁，而定义的全局变量需要手动清理，还会污染命名空间

初始化时调用 mountRef，创建 hook 和 ref 对象

```js
function mountRef(initialValue) {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref; // 创建ref对象。
  return ref;
}
```

更新时调用 updateRef 获取当前 ref

```js
function updateRef(initialValue) {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState; // 取出复用ref对象。
}
```

#### useMemo & useCallback

useMemo 会缓存一个值到 hook 的 memoizedState 属性上，而 useCallback 会缓存一个函数

初始化如下，两者的区别在于 memoizedState 存储的是函数还是值

```ts
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null
): T {
  const hook = mountWorkInProgressHook(); //创建hook
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate(); //计算value
  hook.memoizedState = [nextValue, nextDeps]; //把value和依赖保存在memoizedState中
  return nextValue;
}

function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = mountWorkInProgressHook(); //创建hook
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps]; //把callback和依赖保存在memoizedState中
  return callback;
}
```

更新时如下，逻辑基本一致

```js
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null
): T {
  const hook = updateWorkInProgressHook(); //获取hook
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        //浅比较依赖
        return prevState[0]; //没变 返回之前的状态
      }
    }
  }
  const nextValue = nextCreate(); //有变化重新调用callback
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook(); //获取hook
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        //浅比较依赖
        return prevState[0]; //没变 返回之前的状态
      }
    }
  }

  hook.memoizedState = [callback, nextDeps]; //变了重新将[callback, nextDeps]赋值给memoizedState
  return callback;
}
```

参考：

1. [复盘 react hook 的创造过程](https://github.com/shanggqm/blog/issues/4)
2. [react 源码解析 13.hooks 源码](https://xiaochen1024.com/courseware/60b1b2f6cf10a4003b634718/60b1b374cf10a4003b634725)
