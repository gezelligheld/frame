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

#### 状态派发

#### 处理副作用

#### 状态获取和缓存