#### 类组件中的state

类组件中使用setState更新状态

```js
// callback可以获取当前 setState 更新后的最新 state 的值
setState(obj,callback)
```

触发setState后，react底层主要做了如下的事情

- 首先，setState 会产生当前更新的优先级（老版本用 过期时间expirationTime ，新版本用 优先级lane ）

- 接下来 React 会从 fiber Root 根部 fiber 向下调和子节点，调和阶段将对比发生更新的地方，更新对比 expirationTime ，找到发生更新的组件，合并 state，然后触发 render 函数，得到新的 UI 视图层，完成 render 阶段

- 接下来到 commit 阶段，commit 阶段，替换真实 DOM ，完成此次更新流程

- 接下来会执行 setState 中 callback 函数,如上的()=>{ console.log(this.state.number) }，到此为止完成了一次 setState 全过程

##### 如何限制setState更新视图

- pureComponent 可以对 state 和 props 进行浅比较，如果没有发生变化，那么组件不更新

- shouldComponentUpdate 生命周期可以通过判断前后 state 变化来决定组件需不需要更新，需要更新返回true，否则返回false

##### setState原理

调用 setState 方法，实际上是 React 底层调用 Updater 对象上的 enqueueSetState 方法，enqueueSetState方法创建一个 update ，然后放入当前 fiber 对象的待更新队列中，最后开启调度更新，进入上述讲到的更新流程

```js
enqueueSetState(){
     /* 每一次调用`setState`，react 都会创建一个 update 里面保存了 */
     const update = createUpdate(expirationTime, suspenseConfig);
     /* callback 可以理解为 setState 回调函数，第二个参数 */
     callback && (update.callback = callback) 
     /* enqueueUpdate 把当前的update 传入当前fiber，待更新队列中 */
     enqueueUpdate(fiber, update); 
     /* 开始调度更新 */
     scheduleUpdateOnFiber(fiber, expirationTime);
}
```

那批量更新是什么时候操作的呢？正常 state 更新、UI 交互，都离不开用户的事件，比如点击事件，表单输入等，React 是采用事件合成的形式，每一个事件都是由 React 事件系统统一调度的，那么 State 批量更新正是和事件系统息息相关的

```js
function dispatchEventForLegacyPluginEventSystem(){
    // handleTopLevel 事件处理函数
    batchedEventUpdates(handleTopLevel, bookKeeping);
}

function batchedEventUpdates(fn,a){
    /* 开启批量更新  */
   isBatchingEventUpdates = true;
  try {
    /* 这里执行了的事件处理函数， 比如在一次点击事件中触发setState,那么它将在这个函数内执行 */
    return batchedEventUpdatesImpl(fn, a, b);
  } finally {
    /* try 里面 return 不会影响 finally 执行  */
    /* 完成一次事件，批量更新  */
    isBatchingEventUpdates = false;
  }
}
```

如下面的代码，点击后输出0, 0, 0, callback1 1 ,callback2 1 ,callback3 1

```js
export default class index extends React.Component{
    state = { number:0 }
    handleClick= () => {
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback1', this.state.number)  })
          console.log(this.state.number)
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback2', this.state.number)  })
          console.log(this.state.number)
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback3', this.state.number)  })
          console.log(this.state.number)
    }
    render(){
        return <div>
            { this.state.number }
            <button onClick={ this.handleClick }  >number++</button>
        </div>
    }
}
```

react底层的执行过程如下

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/478aef991b4146c898095b83fe3dc0e7~tplv-k3u1fbpfcp-watermark.image)

但是在异步操作下批量更新的规则会被打破，下面代码输出callback1 1 , 1, callback2 2 , 2,callback3 3 , 3

```js
setTimeout(()=>{
    this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback1', this.state.number)  })
    console.log(this.state.number)
    this.setState({ number:this.state.number + 1 },()=>{    console.log( 'callback2', this.state.number)  })
    console.log(this.state.number)
    this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback3', this.state.number)  })
    console.log(this.state.number)
})
```

react执行过程变成了这样，会触发三次视图渲染

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48e730fc687c4ce087e5c0eab2832273~tplv-k3u1fbpfcp-watermark.image)

要想在异步操作下保持批量更新模式，需要手动开启，开启后仍然输出0, 0, 0, callback1 1 ,callback2 1 ,callback3 1

```js
import ReactDOM from 'react-dom'
const { unstable_batchedUpdates } = ReactDOM

setTimeout(()=>{
    unstable_batchedUpdates(()=>{
        this.setState({ number:this.state.number + 1 })
        console.log(this.state.number)
        this.setState({ number:this.state.number + 1})
        console.log(this.state.number)
        this.setState({ number:this.state.number + 1 })
        console.log(this.state.number) 
    })
})
```

##### 提升更新优先级

react-dom 提供的 flushSync 方法可以将回调函数中的更新任务，放在一个较高的优先级中

如下面所示，flushSync设定了一个高优先级的更新，先渲染一次，打印3；2、4批量更新，渲染一次，打印4；异步操作再渲染一次，打印1

```js
handerClick=()=>{
    setTimeout(()=>{
        this.setState({ number: 1  })
    })
    this.setState({ number: 2  })
    ReactDOM.flushSync(()=>{
        this.setState({ number: 3  })
    })
    this.setState({ number: 4  })
}
render(){
   console.log(this.state.number)
   return ...
}
```

#### 函数组件中的state

函数组件中使用useState更新状态

```js
const [ number , setNumber ] = React.useState(0)
```

上述提到的批量更新和flushSync，函数组件同样适用，只是在当前函数的执行上下文中是获取不到更新后的值的，如下输出0,0,0

```js
const [ number , setNumber ] = React.useState(0)
const handleClick = ()=>{
    ReactDOM.flushSync(()=>{
        setNumber(2) 
        console.log(number);
    })
    setNumber(1) 
    console.log(number)
    setTimeout(()=>{
        setNumber(3) 
        console.log(number)
    })   
}
```

原因是因为函数组件的更新其实就是函数的执行，一次执行过程中函数内部所有变量重新声明，所改变的state只有在下一次函数组件执行时才会被更新

##### useState和setState的异同

- 相同：setState和 useState 更新视图，底层都调用了 scheduleUpdateOnFiber 方法，而且事件驱动情况下都有批量更新规则

- 不同：

    - setState 不会浅比较两次 state 的值，只要调用 setState，在没有其他优化手段的前提下，就会执行更新。但是 useState 中的 dispatchAction 会默认比较两次 state 是否相同，然后决定是否更新组件

    - setState 有专门监听 state 变化的回调函数 callback，可以获取最新state；但是在函数组件中，只能通过 useEffect 来执行 state 变化引起的副作用