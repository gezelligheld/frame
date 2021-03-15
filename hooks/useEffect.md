#### 使用

- 给函数组件增加了操作副作用的能力，它跟class组件中的componentDidMount、componentDidUpdate和componentWillUnmount具有相同的用途，只不过被合并成了一个 API

> 副作用：函数式编程中的概念，当调用函数时，除了返回值之外，还对主调用函数产生附加的影响。例如修改全局变量（函数外的变量）或修改参数

```js
import React,{useState,useEffect} from 'react'

function userState(){
    useEffect(()=>{
        //会在组件挂在时和组件更新时调用，即componentDidMount和componentDidUpdate
        //第二个参数是一个数组，存放该hook所依赖的state和函数，只有相关依赖改变时才触发componentDidUpdate重新渲染页面
        //当第二个参数为空数组或不写时，不会触发componentDidUpdate
        return ()=>{
            //组件卸载时触发该回调，相当于componentWillUnmount
        }
    },[name])
    return (
        <div></div>
    )
}

export default userState
```

- 可以将不同的业务逻辑分离到多个useEffect中，顺序执行

#### 原理

挂载时创建一个hook对象，用单向链表存储每个effect，将effectQueue绑定到fiberNode上，每次渲染完成后依次执行队列中存储的effect

```js
type Effect{
    tag: any,           // 用来标识effect的类型，
    create: any,        // 副作用函数
    destroy: any,       // 取消副作用的函数，
    deps: Array,        // 依赖
    next: Effect,       // 循环链表指针
}

type EffectQueue{
    lastEffect: Effect
}

type FiberNode{
    memoizedState:any  // 用来存放某个组件内所有的Hook状态
    updateQueue: any  
}
```

实现如下

```js
let componentUpdateQueue = null;
function pushEffect(tag, create, deps){
    // 构建更新队列
    // ...
}

function useEffect(create, deps){
    if(isMount)(
        mountEffect(create, deps)
    )else{
        updateEffect(create, deps)
    }
}

function mountEffect(create, deps){
    const hook = createHook();
    hook.memoizedState = pushEffect(xxxTag, create, deps);
}

function updateEffect(create, deps){
    const hook = getHook();
    if(currentHook!==null){
        const prevEffect = currentHook.memoizedState;
        if(deps!==null){
            // 浅比较更新前后的deps是否发生了变化
            if(areHookInputsEqual(deps, prevEffect.deps)){
                pushEffect(xxxTag, create, deps);
                return;
            }
        }
    }
    
    hook.memoizedState = pushEffect(xxxTag, create, deps);
}
```

> useEffect触发后会把卸载函数（return后）存起来，下一次再次触发时先执行卸载，再创建effect

#### 模拟实现

```js
// 上一次的依赖，是个二维数组
const lastDepsArr = [];
// 上一次effect的卸载函数
const lastClearCallbacks = [];
let index = 0;

function useEffect(callback, deps) {
    const lastDeps = lastDepsArr[index];
    if (!lastDeps //初次渲染
        || !deps // 没有添加依赖
        || deps.some((dep, index) => !Object.is(dep, lastDeps[index])) // 存在一个依赖项与上次render的依赖不一致
    ) {
        lastDepsArr[index] = deps;
        // 执行当前的effect时，需要先执行上一次effect的卸载函数
        if (lastClearCallbacks[index]) {
            lastClearCallbacks[index]();
        }
        lastClearCallbacks[index] = callback();
        callback();
    }
    index ++;
}
```