# react 总结

##### react vs vue

- 数据模型不同。react 最多算是一个 MV 架构，通过状态驱动视图，强调数据不可变性，如果一个引用类型的变化想引起视图的变化，需要改变其引用；vue 是 MVVM 架构，通过 Proxy 作数据劫持实现数据绑定，内部实现了 ViewModel

- 更新渲染的逻辑不同。vue 每个组件都有一个监视器，以组件的粒度更新视图；react 会遍历整个 fiber 树，找到其中的副作用然后再更新视图，concurrent 模式下捕捉浏览器每一帧中的空闲时间执行更新任务，防止 js 执行太久造成页面卡顿

- 生态不同。vue 的路由库和状态管理库都由官方维护支持，react 则将这些问题交给社区，因此 react 的社区生态比 vue 更加繁荣一些，这也就需要 react 项目的规范需要更完善一些，防止一个项目内不同开发人员代码风格差异过大，难以维护

##### jsx

jsx 是描述 dom 结构和信息的对象，让开发者从 dom 操作中解放出来，以声明式的方式渲染 UI。jsx 相比于直接操作 dom 多了一层内存运算，初次渲染比直接操作 dom 要慢，状态更新时会进行 diff，尽量复用之前的节点，减少了一些重绘重排

##### state

不管是 useState 还是 setState，在异步回调中执行时会打破批量更新的规则，变成了同步的过程，执行一次 useState/setState 就会触发一次调和更新。区别在于，setState 执行完毕后会在回调中拿到最新的值，且每调用一次就会发起更新流程；而 useState 会做浅比较，一个更新周期结束后才能拿到最新的值

##### props（render props、插槽组件、React.Children）

##### ref（ref 转发、缓存数据、获取 dom）

##### 生命周期

- 初始化时，首先执行 constructor，可以初始化 state 等操作，然后如果有 getDriviedStateFromProps 就执行它，用来将接收到的 props 派生成 state 添加到当前组件实例中，否则执行 componentWillMount。然后执行 render 函数，触发 render 流程，到了 commit 流程时触发 componentDidMount，用来处理啊副作用

- 更新时，如果是 props 引起的更新，如果有 getDerivedStateFromProps 触发其派生出新的状态，否则触发 componentWillReceiveProps。然后执行 shouldComponentUpdate，如果返回 false 则停止本次更新流程，否则继续更新流程。然后触发 componentWilUpdate，执行 render 函数，触发 render 流程，到了 commit 阶段触发 componentDidUpdate

- 其他生命周期。组件销毁时执行 componentWillUnmount，触发于 commit 阶段，用来清除定时器、解绑事件等；组件捕获到错误时触发 componentDidCatch，触发于 commit 阶段，用来上报错误日志、显示相应 ui 等

> componentWillMount、componentWillReceiveProp、componentWilUpdate 都被打上了 unsafe 的标志，原因是这些生命周期是在 render 之前执行的，可能会执行多次

##### 渲染优化策略（useMemo、shouldComponentUpdate、PureComponent、memo）

除了 react 自身做了大量渲染优化策略外，如批量更新、时间分片、diff 算法，开发者主要可以做两方面的优化：父组件阻止子组件重新渲染和子组件阻止自身重新渲染。主要策略如下：当父组件状态更新，如果缓存了 element 对象，则子组件不进入更新流程，否则继续向下调和子节点，当采用了 memo 就会对 props 做浅比较。如果是函数组件，可以使用 PureComponent 对 props 做浅比较，使用 shouldComponentUpdate 决定是否进行更新

##### hook（设计初衷、常用 hook、函数组件生命周期替代、自定义 hook、hook 原理、useEffect 和 useLayoutEffect 异同）

16.8 版本之前函数组件是无状态的，类组件状态难以复用，逻辑难以分离，借助高阶组件需要嵌套，比较难以维护；之后 hook 的出现使得函数组件拥有了状态和改变状态的能力，自定义 hook 使得状态复用和逻辑分离变得轻易起来

函数组件的多个 hook 形成链表保存到组件对应的 fiber 上，按照 hook 调用的顺序去索引对应的 hook，限制了 hook 不能在 if 语句中执行，否则会由于 if 条件的变动前后索引到不同的 hook 导致错乱

使函数组件拥有状态和改变状态的能力依靠 useState 和 useReducer 这两个 hook，useReducer 可以在一组关联的状态和动作下使用；useEffect 和 useLayoutEffect 可以处理副作用，它们都在 commit 阶段执行上一次的卸载函数和本次的回调函数，不同的是，useEffect 适合处理异步请求，其回调异步处理，而 useLayoutEffect 适合操作 dom，其回调同步处理，会阻塞视图渲染；useRef 和 useImperativeHandle 搭配 forwardRef 可以进行 ref 转发，将自身状态或方法抛给外部使用；useCallback 和 useMemo 可以将值和函数缓存在 fiber 上，可以减少不必要的重复渲染

##### 事件系统（事件合成、事件触发流程、模拟捕获和冒泡）

为了兼容不同的浏览器和方便统一管理，react 有自己的一套事件系统。元素上绑定的事件最终都绑定到了 app 容器上，react 事件可能是多个原生事件合成的事件，当遍历 fiber 树的时候，会将捕获阶段的事件添加到数组头部，冒泡阶段的事件添加到数组尾部，直到顶层元素，然后依次执行

##### concurrent（fiber、Scheduler 时间分片、调度优先级、更新任务的暂停和继续）

16 版本之前当状态变更的时候，会从根节点开始递归更新，如果项目比较复杂这一过程会比较耗时；fiber 的出现使得更新任务可以拆分，不至于长时间阻塞 GUI 渲染线程造成页面卡顿，可以在浏览器空闲时间执行更新任务。高优先级任务可以打断低优先级任务，让页面在繁重的更新流程下也能保持良好的交互

以 60hz 的屏幕刷新率为例，一帧就是 16.6ms，浏览器在一帧内会依次处理 input 输入等的交互、requestAnimationFrame、渲染，剩余的时间就是空闲时间可以用来执行更新任务。react 内部使用 MessageChannel 捕捉这个空闲时间，这就是所谓的时间分片

区别于 legacy 模式，concurrent 模式在进入调和流程之前先进行调度，优先处理高优先级任务，这里的任务其实就是某个更新周期的 render 阶段，然后获取空闲帧去执行，而 legacy 模式会按顺序执行更新任务，没有优先级之分

> vue 没有时间分片的概念，每个组件对应一个监视器，在一次更新中 vue 能够快速响应，以组件的粒度更新组件

##### 17 版本调和流程

更新 fiber 并最终修改页面的过程称为调和，主要分为以下两个阶段

- render 阶段用来构建 fiber 树和生成副作用链表。初始化时，首先创建作为 react 应用根基的 fiberRoot 节点以及通过 ReactDOM.render 创建的 rootFiber 节点。正在内存中构建的 Fiber 树称为 workInProgress Fiber 树，根据 jsx 构建 workInProgress Fiber 树，最后切换成 current Fiber 树用来渲染视图；更新时，首先向下遍历 current Fiber 树，与最新的 jsx 经过 diff 算法构建 workInProgress Fiber 树，可以复用上一次渲染的节点，也可以减少 dom 操作，同时追踪副作用并打上 tag，然后向上回溯时将这些标识为带有副作用的节点添加到链表中。这样复用上一次的 current fiber 树作为缓存树生成 workInProgress 过程称为双缓冲树

- commit 阶段主要用来根据 render 阶段所给的副作用链表进行 dom 节点的增删改，最终渲染到页面上

其中利用 current 树和最新的 jsx 构建 workInProgress 树时，使用了 diff 算法，使用了一些策略将复杂度由 n^3 降低为 n

- tree：只对同一层级的节点进行比较，如果某个节点不存在不会再处理子节点，对不同层级的节点只有删除和创建操作

- component：如果遇到同一类型的组件才会进行 tree diff 和 element diff，否则只有删除和创建操作

- element：同一层级的节点依赖唯一的 key 值进行位置变换，设置两个指针 lastIndex 和 nextIndex，初始都为 0，nextIndex 每次自增 1，如果节点之前的索引大于 lastIndex，则将该索引赋值给 lastIndex，否则移动该节点到 nextIndex 位置

##### 其他（context、suspense、lazy、router）

##### 状态管理（redux 工作流程、原理、中间件，mobx，差别和适用场景）

- redux：基于发布订阅，会创建一个表示状态信息的 store，可以通过 dispatch 修改 store，ui 组件可以通过 subscribe 订阅 store 的变化。默认只支持同步修改 store，可以通过 redux-thunk 等中间件增强 dispatch，使其可以处理其他副作用。redux 一般适用于大型项目，由于 store 全局唯一并且使用纯函数修改状态使得方便调试和时间旅行，但是需要书写太多模板代码

- mobx：基于观察者模式，使用 observable 给属性绑定状态，这些属性就是观察者，实例化的时候执行了 makeObservable 建立响应式，内部创建了一个观察者管理者，遍历观察者属性依次使用 Object.defineProperty 设置为访问器属性，触发 set 时收集依赖，触发 get 时通知依赖作出相应动作。mobx 适合小型项目的全局状态管理，使用起来比较简单，但不方便时间旅行，数据流向比较多样。mobx 更适合接管局部的状态，利用面向对象的方式将业务逻辑和视图分离
