#### Fiber的更新流程

1. render阶段

代码中写的 JSX，通过 Babel 或 TS 的处理后，会转化为 React.createElement，进一步转化为 Fiber 树，Fiber 树是链表结构

![image](https://pic3.zhimg.com/80/v2-ceaa7a28719d33ddd706f2313c813742_1440w.jpg)

无论是类组件、函数组件或宿主组件（div等），在底层都会统一抽象为 Fiber 节点 ，拥有父节点（return）、子节点（child）或者兄弟节点（sibling）的引用，方便对于 Fiber 树的遍历，同时组件与 Fiber 节点会建立唯一映射关系

当使用setState更新状态时，找到组件对应的Fiber节点，在其updateQueue属性中插入一个 update 对象，updateQueue 也是一个链表结构，会记录所属 Fiber 节点上收集到的更新，然后从触发setState的节点向上回溯，通知沿途的Fiber节点有子孙节点更新了，直到顶层的HostRoot

然后从 HostRoot 开始对 Fiber 树进行深度优先遍历。每个 Fiber 节点在遍历到时，若自身存在变更，会根据 Fiber 类型对节点执行创建/更新，其中包含了执行部分生命周期，给 Fiber 节点打上 effectTag 等操作。effectTag 代表了 Fiber 节点做了怎样的变更，具有 effectTag 的 Fiber 会成为 effect。

每个 Fiber 中带有自身子节点的信息，据此来判断是否需要继续向下深度遍历，若无需再向下遍历，判断兄弟节点是否需要更新，如果没有则回溯到父节点，并将自身及子树上的effect传递给父节点，直到HostRoot

2. commit阶段

HostRoot完成所有的effect收集，代表着本次更新需要进行哪些变更，最后将这些更新提交给宿主环境，也就是更新DOM