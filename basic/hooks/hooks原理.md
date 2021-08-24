#### 设计初衷

react组件复用，经历了mixins -> 高阶组件HOC -> 自定义hook的过程，类组件状态难以复用，借助高阶组件嵌套层级过多，逻辑复杂，难以维护

16.8版本之前，组件分为两种，含有状态的类组件和无状态的函数组件

如果想要去class改造函数组件的话，函数组件需要拥有和类组件一样的能力，即

- class组件可以存储实例状态，挂到this上，而函数组件每次执行后，状态都会被初始化

- class组件拥有setState方法改变状态，而函数组件没有

而hooks的出现，使函数组件也拥有了维持状态、改变状态的能力，自定义hook也使得逻辑复用更加容易

#### hook和fiber

类组件的状态如state、props、context本质上是存在类组件对应的fiber上，hooks既然赋予了函数组件如上的能力，也离不开函数组件对应的fiber

hooks 对象主要有三种处理策略

1. ContextOnlyDispatcher：防止在函数组件外调用hooks，否则抛出异常

2. HooksDispatcherOnMount：函数组件初始化时，初始化hooks，建立起其fiber和hooks的关系

3. HooksDispatcherOnUpdate：函数组件更新时，需要hooks去获取或更新状态

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

##### 函数组件触发

在fiber调和的过程中，遇到函数组件类型的fiber，会调用updateFunctionComponent中的renderWithHooks更新fiber

```js
let currentlyRenderingFiber
function renderWithHooks(current,workInProgress,Component,props){
    currentlyRenderingFiber = workInProgress;
    workInProgress.memoizedState = null; /* 每一次执行函数组件之前，先清空状态 （用于存放hooks列表）*/
    workInProgress.updateQueue = null;    /* 清空状态（用于存放effect list） */
    ReactCurrentDispatcher.current =  current === null || current.memoizedState === null ? HooksDispatcherOnMount : HooksDispatcherOnUpdate /* 判断是初始化组件还是更新组件 */
    let children = Component(props, secondArg); /* 执行我们真正函数组件，所有的hooks将依次执行。 */
    ReactCurrentDispatcher.current = ContextOnlyDispatcher; /* 将hooks变成第一种，防止hooks在函数组件外部调用，调用直接报错。 */
}
```

其中

- workInProgress是正在调和的当前函数组件对应的fiber树
- 对于类组件，memoizedState存储的是state信息，对于函数组件memoizedState存储的是hooks信息
- 对于函数组件，updateQueue用于存放useEffect/useLayoutEffect 产生的effect list，用于在最终的commit阶段执行副作用
- 引用的 React hooks都是从 ReactCurrentDispatcher.current 中的， React 就是通过赋予 current 不同的 hooks 对象达到监控 hooks 是否在函数组件内部调用
- 真正的函数组件Component也在此时执行，里面的每个hooks也依次执行

##### 初始化hook

上面提到的初始化hooks时，会建立fiber与hooks的关系，通过mountWorkInProgressHook实现

```js
function mountWorkInProgressHook() {
  const hook = {  memoizedState: null, baseState: null, baseQueue: null,queue: null, next: null,};
  if (workInProgressHook === null) {  // 只有一个 hooks
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {  // 有多个 hooks
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

如下代码，最终初始化的结果将会是这样

```js
export default function Index(){
    const [ number,setNumber ] = React.useState(0) // 第一个hooks
    const [ num, setNum ] = React.useState(1)      // 第二个hooks
    const dom = React.useRef(null)                 // 第三个hooks
    React.useEffect(()=>{                          // 第四个hooks
        console.log(dom.current)
    },[])
    return <div ref={dom} >
        <div onClick={()=> setNumber(number + 1 ) } > { number } </div>
        <div onClick={()=> setNum(num + 1) } > { num }</div>
    </div>
}
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b589f284235c477e9e987460862cc5ef~tplv-k3u1fbpfcp-watermark.awebp)

##### hook更新

首先取出 workInProgres.alternate 里面对应的 hook ，然后根据之前的 hooks 复制一份，形成新的 hooks 链表关系

然后解释一下为什么不能在条件语句中使用hook，假如使用了

```js
export default function Index({ showNumber }){
    let number, setNumber
    showNumber && ([ number,setNumber ] = React.useState(0)) // 第一个hooks
    const [ num, setNum ] = React.useState(1)      // 第二个hooks
    const dom = React.useRef(null)                 // 第三个hooks
    React.useEffect(()=>{                          // 第四个hooks
        console.log(dom.current)
    },[])
    return <div ref={dom} >
        <div onClick={()=> setNumber(number + 1 ) } > { number } </div>
        <div onClick={()=> setNum(num + 1) } > { num }</div>
    </div>
}
```

当变量showNumber前后变化不一致时，复用workInProgres.alternate 里面对应的 hook会发现hook类型不一样，抛出错误

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1a23945ff0748d585e99786b4dff12e~tplv-k3u1fbpfcp-watermark.awebp)

#### 状态派发

useState和useReducer让函数组件拥有了状态，其内部实现都是dispatchAction函数

初始化useState过程入下

```js
function mountState(initialState){
     const hook = mountWorkInProgressHook();
    if (typeof initialState === 'function') {initialState = initialState() } // 如果 useState 第一个参数为函数，执行函数得到初始化state
     hook.memoizedState = hook.baseState = initialState;
    const queue = (hook.queue = { ... }); // 负责记录更新的各种状态。
    const dispatch = (queue.dispatch = (dispatchAction.bind(  null,currentlyRenderingFiber,queue, ))) // dispatchAction 为更新调度的主要函数 
    return [hook.memoizedState, dispatch];
}
```

每次改变state，其实调用的是dispatchAction

- 首先创建一个update，将其放入待更新的队列pending中

- 当前fiber没有更新任务时，将这一次的state和上一次的state进行浅比较，如果相同退出更新，否则发起更新调度任务

```js
function dispatchAction(fiber, queue, action){
    /* 第一步：创建一个 update */
    const update = { ... }
    const pending = queue.pending;
    if (pending === null) {  /* 第一个待更新任务 */
        update.next = update;
    } else {  /* 已经有带更新任务 */
       update.next = pending.next;
       pending.next = update;
    }
    if( fiber === currentlyRenderingFiber ){
        /* 说明当前fiber正在发生调和渲染更新，那么不需要更新 */
    }else{
       if(fiber.expirationTime === NoWork && (alternate === null || alternate.expirationTime === NoWork)){
            const lastRenderedReducer = queue.lastRenderedReducer;
            const currentState = queue.lastRenderedState;                 /* 上一次的state */
            const eagerState = lastRenderedReducer(currentState, action); /* 这一次新的state */
            if (is(eagerState, currentState)) {                           /* 如果每一个都改变相同的state，那么组件不更新 */
               return 
            }
       }
       scheduleUpdateOnFiber(fiber, expirationTime);    /* 发起调度更新 */
    }
}
```

如下面的代码

```js
export default  function Index(){
    const [ number , setNumber ] = useState(0)
    const handleClick=()=>{
        setNumber(num=> num + 1 ) // num = 1
        setNumber(num=> num + 2 ) // num = 3 
        setNumber(num=> num + 3 ) // num = 6
    }
    return <div>
        <button onClick={() => handleClick() } >点击 { number } </button>
    </div>
}
```

点击一次按钮，触发了三次dispatchAction，会存到pending队列中，被合并成一次更新；当再次执行setNumber时，会触发更新hooks逻辑，调用updateReducer，会将待更新队列拿出来，合并成最新的值给到memoizedState，这样再次调用setNumber时内部可以拿到最新值

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/750ee5e50ff8494791f52bd095b305ca~tplv-k3u1fbpfcp-watermark.awebp)

#### 处理副作用

fiber更新的render阶段是没有任何副作用的，想要操作的类型被打上了不同的effectTag，在commit阶段统一处理副作用

对于类组件，有componentDidMount/componentDidUpdate 固定的生命周期钩子，用于执行初始化/更新的副作用逻辑，但是对于函数组件，可能存在多个 useEffect/useLayoutEffect ，hooks 把这些 effect，独立形成链表结构，在 commit 阶段统一处理和执行

```js
function mountEffect(create,deps){
    const hook = mountWorkInProgressHook();
    const nextDeps = deps === undefined ? null : deps;
    currentlyRenderingFiber.effectTag |= UpdateEffect | PassiveEffect;
    // 通过 pushEffect 创建一个 effect，并保存到当前 hooks 的 memoizedState 属性
    // 如果存在多个 effect 或者 layoutEffect 会形成一个副作用链表，绑定在函数组件 fiber 的 updateQueue 上
    hook.memoizedState = pushEffect( 
      HookHasEffect | hookEffectTag, 
      create, // useEffect 第一次参数，就是副作用函数
      undefined, 
      nextDeps, // useEffect 第二次参数，deps    
    )
}
```

如下代码

```js
React.useEffect(()=>{
    console.log('第一个effect')
},[ props.a ])
React.useLayoutEffect(()=>{
    console.log('第二个effect')
},[])
React.useEffect(()=>{
    console.log('第三个effect')
    return () => {}
},[])
```

副作用链表如下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21485f1321864045a73bca1b3afdc948~tplv-k3u1fbpfcp-watermark.awebp)

当依赖项发生变化时，会更新副作用链表，同时打上effectTag，在commit阶段通过effectTag识别是useEffect 还是 useLayoutEffect的副作用，然后同步处理useLayoutEffect，异步处理useEffect

```js
function updateEffect(create,deps){
    const hook = updateWorkInProgressHook();
    if (areHookInputsEqual(nextDeps, prevDeps)) { /* 如果deps项没有发生变化，那么更新effect list就可以了，无须设置 HookHasEffect */
        pushEffect(hookEffectTag, create, destroy, nextDeps);
        return;
    } 
    /* 如果deps依赖项发生改变，赋予 effectTag ，在commit节点，就会再次执行我们的effect  */
    currentlyRenderingFiber.effectTag |= fiberEffectTag
    hook.memoizedState = pushEffect(HookHasEffect | hookEffectTag,create,destroy,nextDeps)
}
```

#### 状态获取和缓存

- useRef

useRef创建并维护一个ref对象，用于获取dom、保存一些状态等，一些不希望引起页面重新渲染的值可以存到ref中；另外全局变量也可以用ref代替，ref会随着组件的销毁而销毁，而定义的全局变量需要手动清理，还会污染命名空间

```js
// 创建
function mountRef(initialValue) {
  const hook = mountWorkInProgressHook();
  const ref = {current: initialValue};
  hook.memoizedState = ref; // 创建ref对象。
  return ref;
}

// 更新
function updateRef(initialValue){
  const hook = updateWorkInProgressHook()
  return hook.memoizedState // 取出复用ref对象。
}
```

- useMemo

useMemo会缓存一个值到hook的memoizedState属性上，更新时会对比依赖项是否发生变化，没有直接返回缓存值，有则执行第一个函数参数生成新的缓存值

```js
// 创建
function mountMemo(nextCreate,deps){
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

// 更新
function updateMemo(nextCreate,deps){
    const hook = updateWorkInProgressHook();
    const prevState = hook.memoizedState; 
    const prevDeps = prevState[1]; // 之前保存的 deps 值
    if (areHookInputsEqual(nextDeps, prevDeps)) { //判断两次 deps 值
        return prevState[0];
    }
    const nextValue = nextCreate(); // 如果deps，发生改变，重新执行
    hook.memoizedState = [nextValue, nextDeps];
    return nextValue;
}
```

参考：
1. [复盘react hook的创造过程](https://github.com/shanggqm/blog/issues/4)