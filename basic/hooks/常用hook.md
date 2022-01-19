#### useState

让函数组件拥有state状态值和更新state的函数

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
```

> setState函数的标识是稳定的，所以不需要作为useEffect、useCallback、useMemo的依赖

与class组件中的setState方法不同，useState不会自动合并更新对象，可以用以下方式达到更新合并的效果

```js
setState(prevState => {
  // 也可以使用Object.assign
  return {...prevState, ...updatedValues};
});
```
函数式更新，即新的state需要先前的state计算得出，可以采用以下形式

```js
setState(prevState => preState + 1);
```

模拟实现一个useState

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

#### useEffect

给函数组件增加了操作副作用的能力，它跟class组件中的componentDidMount、componentDidUpdate和componentWillUnmount类似

> 副作用：函数式编程中的概念，当调用函数时，除了返回值之外，还对主调用函数产生附加的影响。例如修改全局变量（函数外的变量）或修改参数

```js
import React,{useState,useEffect} from 'react'

function userState(){
    useEffect(()=>{
        // 依赖项变化时执行
        return ()=>{
            // 组件卸载时触发该回调，相当于componentWillUnmount
            // 依赖项发生变化时，会执行上一次的卸载函数和这一次的effect
        }
    },[name]);

    useEffect(() => {
        // 第二个参数不写时，每次rerender都会执行
    });

    return (
        <div></div>
    )
}
```

可以将不同的业务逻辑分离到多个useEffect中，顺序执行

模拟实现一个useEffect

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

#### useMemo和useCallback

将值或函数缓存，避免每次render（即函数组件执行）后重新生成，并随依赖项的变化而变化

```js
const data = useMemo(() => {
    // todo
}, [deps]);

const fn = useCallback(() => {
    // todo
}, [deps]);
```

使用useMemo和useCallback可以避免一些重复渲染，提高性能，但本身也存在开销，故个人认为在以下几种场景适用

- 值或函数运行的开销较大

- 值或函数依赖了某些状态，且需要在useEffect/useLayoutEffect中执行时，或作为props或context传递时，或作为自定义hook的结果暴露时，避免重复触发

#### useRef

用途有三（大概）：

- 获取dom

- 作为一个存储值的容器，用来定义一些无需引起渲染的值，ref的值在一个生命周期内是不变的，可以代替全局变量，一方面防止污染全局命名空间，另一方面ref会随组件的销毁而自动清除，便于垃圾回收

- ref转发，子组件通过useImperativeHandle将子组件的值或方法传递给父组件，父组件通过ref获取

#### useReducer

useReducer是除useState以外，官方提供的另一种状态管理的方式，可以理解为局部的redux，将一组关联的状态和动作进行管理，一定程度上实现业务逻辑的抽离，降低耦合

```js
// 第一个参数：应用的初始化
const initialState = {count: 0};

// 第二个参数：state的reducer处理函数
function reducer(state, action) {
    switch (action.type) {
        case 'increment':
            return {count: state.count + 1};
        case 'decrement':
            return {count: state.count - 1};
        default:
            throw new Error();
    }
}

function Counter() {
    // 返回值：最新的state和dispatch函数
    const [state, dispatch] = useReducer(reducer, initialState);
    return (
        <>
            // useReducer会根据dispatch的action，返回最终的state，并触发rerender
            Count: {state.count}
            // dispatch 用来接收一个 action参数「reducer中的action」，用来触发reducer函数，更新最新的状态
            <button onClick={() => dispatch({type: 'increment'})}>+</button>
            <button onClick={() => dispatch({type: 'decrement'})}>-</button>
        </>
    );
}
```

#### useContext

当一个状态需要下发到多个子组件或需要层层传递时，可以定义局部的上下文

```js
// 父组件
import { createContext } from 'react';

const Context = createContext<{ data: any[] }>({
  data: []
});

function Demo() {
    const [userList, setUserList] = useState([]);

    return (
        <div>
            <Context.Provider value={{data: userList}}>
                <Child1 />
                <Child2 />
                ...
            <Context.Provider />
        </div>
    );
}
```

```js
// 子组件
import context from './context';

function Child1() {
    const {data} = useContext(context);
    // ...
}
```

#### useLayoutEffect

区别于useEffect，useLayoutEffect是同步执行的，会阻塞渲染，最常用的场景是在这里进行DOM相关的操作，如果放在useEffect中会引起页面闪烁

```js
useLayoutEffect(() => {
    // todo
}, []);
```

#### useImperativeHandle

useImperativeHandle接受forwardRef转发的父组件标识的ref，将第二个参数的返回值注入到ref中，供父组件获取子组件内的一些数据

```js
// 父组件
function Parent() {
    const sonRef = useRef(null);
    console.log(sonRef.current);
    return (
        <Son ref={sonRef} />
    )
}
```

```js
// 子组件
function Son(props,ref) {
    const [ inputValue , setInputValue ] = useState('')
    useImperativeHandle(ref,() => ({inputValue}),[inputValue])
    return <div>
        <input placeholder="请输入内容"  value={inputValue} />
    </div>
}
const ForwarSon = forwardRef(Son);
```