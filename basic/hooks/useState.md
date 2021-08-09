#### 使用

- 让函数组件拥有state状态值和更新state的函数

```js
import React,{useState,useEffect} from 'react'
function userState(){
    //useState方法返回一个数组，解构获取
    //数组的第一项是state的名字，数组第二项是修改state的方法，括号内为默认值
    const [name,setName]=useState("abc")
    const [age,setAge]=useState("10")

    return (
        <div>
            {name}:{age}
            <button onClick={()=>{
                setName("cba")
                setAge("11")
            }></button>
        </div>
    )
}
export default userState
```

> setState函数的标识是稳定的，所以不需要作为useEffect、useCallback、useMemo的依赖

- 与class组件中的setState方法不同，useState不会自动合并更新对象，可以用以下方式达到更新合并的效果

```js
setState(prevState => {
  // 也可以使用Object.assign
  return {...prevState, ...updatedValues};
});
```
- 函数式更新，即新的state需要先前的state计算得出，可以采用以下形式

```js
setState(prevState => preState + 1);
```

#### 原理

useState简易实现如下，利用闭包存储状态

```js
function useState(initialState) {
    let state = initialState;
    function dispatch(newState) {
        state = newState;
    }
    return [state, dispatch];
}
```

但是并不满足，为了在函数组件在初始化、状态变更时调用useState可以获取到正确的值，需要一个数据结构存储上一次和这一次的state

```js
type Hook = {
    memoizedState: any,   // 上一次完整更新之后的最终状态值
    queue: UpdateQueue<any, any> | null, //更新队列
};
```

改进如下

```js
function createNewHook(){
    return {
        memoizedState: null,
        baseUpdate: null
    };
}

function dispatchAction(action){
    // 使用数据结构存储所有的更新行为，以便在rerender流程中计算最新的状态值
    storeUpdateActions(action);
    // 执行fiber的渲染
    scheduleWork();
}

function useState(initialState){
    if(isMounting){
        return mountState(initialState);
    }
    
    if(isUpdateing){
        return updateState(initialState);
    }
}

// 第一次调用组件的useState时实际调用的方法
function mountState(initialState){
    let hook = createNewHook();
    hook.memoizedState = initalState;
    return [hook.memoizedState, dispatchAction]
}

// 第一次之后每一次执行useState时实际调用的方法
function updateState(initialState){
    // 根据dispatchAction中存储的更新行为计算出新的状态值，并返回给组件
    doReducerWork();
    
    return [hook.memoizedState, dispatchAction];
}
```

仍然存在问题，初始化和状态变更调用的两个函数如何共享hook对象，先看一个问题

如下，在同一个事件回调中进行了多次dispatch

```js
function Count(){
    const [count, setCount] = useState(0);
    const [countTime, setCountTime] = useState(null);
    
    function clickHandler(){
        // 调用多次dispatchAction
        setCount(1);
        setCount(2);
        setCount(3);
        //...
        setCountTime(Date.now())
    }
    
    return (
        <div>
            <div>{count} in {countTime}</div>
            <button onClick={clickHandler} >update counter</button>
        </div>
    );
}
```

我们并不希望去渲染3次，需要同步执行完所有dispath后，将dispatch的state顺序存储，使用一个队列存储，用单向链表存储这些更新队列

```js
type Queue{
    last: Update,   // 最后一次更新逻辑
    dispatch: any,
    lastRenderedState: any  // 最后一次渲染组件时的状态
}

type Update{
    action: any,    // 状态值
    next: Update    // 下一次Update
}
```

改进如下

```js
function mountState(initialState){
    let hook = createNewHook();
    hook.memoizedState = initalState;
    
    // 新建一个队列
    const queue = (hook.queue = {
        last: null,
        dispatch: null,
        lastRenderedState:null
    });
    
    //通过闭包的方式，实现队列在不同函数中的共享。前提是每次用的dispatch函数是同一个
    const dispatch = dispatchAction.bind(null, queue);
    queue.dispath = dispatch;
    return [hook.memoizedState, dispatch]
}

function dispatchAction(queue, action){
    // 使用数据结构存储所有的更新行为，以便在rerender流程中计算最新的状态值
    const update = {
        action,
        next: null
    }
    
    let last = queue.last;
    if(last === null){
        update.next = update;
    }
    else{
        // ... 更新循环链表
    }
    
    // 执行fiber的渲染
    scheduleWork();
}

function updateState(initialState){
    // 获取当前正在工作中的hook
    const hook = updateWorkInProgressHook();
    
    // 根据dispatchAction中存储的更新行为计算出新的状态值，并返回给组件
    (function doReducerWork(){
        let newState = null;
        do{
            // 循环链表，执行每一次更新
        }while(...)
        hook.memoizedState = newState;
    })();
     
    return [hook.memoizedState, hook.queue.dispatch];
}
```

#### 模拟实现

```js
let index = 0;
const states = [];

function useState(initialState) {
    states[index] = states[index] || initialState;
    function dispatch(newState) {
        states[index] = newState;
    }
    index ++;
    return [states[index], dispatch];
}
```

参考文档：
1. [手写简单的useState/useEffect](https://zhuanlan.zhihu.com/p/265662126)