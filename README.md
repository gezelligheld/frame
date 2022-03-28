# react 总结

- basic 基础

- deep 原理

- ecology 生态

#### 知识点

- jsx（虚拟 dom 性能、本质）
- state（this.setState 同步异步、批量更新、this.setState 和 useState 异同）
- props（render props、插槽组件、React.Children）
- ref（ref 转发、缓存数据、获取 dom）
- 生命周期（挂载 constructor、getDerivedStateFromProps、componentWillMount、render、componentDidMount，更新 componentWillReceiveProps/getDerivedStateFromProp、shouldComponentUpdate、componentWillUpdate、render、getSnapshotBeforeUpdate、componentDidUpdate，销毁 componentWillUnMount，错误 componentDidCatch、getDerivedStateFromError）
- 渲染优化策略（useMemo、shouldComponentUpdate、PureComponent、memo）
- hook（设计初衷、常用 hook、函数组件生命周期替代、自定义 hook、hook 原理、useEffect 和 useLayoutEffect 异同）
- 事件系统（事件合成、事件触发流程、模拟捕获和冒泡）
- fiber（设计初衷，render 阶段 构建 fiber 树、生成 effectlist，commit 阶段 操作 dom，diff 时间复杂度优化策略、tree、component、element）
- concurrent（Scheduler 时间分片、调度优先级、更新任务的暂停和继续，优先级 lane）
- 其他（context、suspense、lazy、router）
- 状态管理（redux 工作流程、原理、中间件，mobx，差别和适用场景）
