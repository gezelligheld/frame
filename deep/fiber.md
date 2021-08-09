#### stack reconciler

v16之前，渲染核心算法称之为Stack reconciler

react在进行组件渲染时，从setState开始到渲染完成整个过程是同步的，如果需要渲染的组件比较庞大，js执行会占据主线程时间较长，会导致页面响应度变差，出现卡顿

Stack reconciler的工作流程很像函数的调用过程。父组件里调子组件，可以类比为函数的递归。在setState后，react会立即开始reconciliation过程，从父节点（Virtual DOM）开始遍历，以找出不同。将所有的Virtual DOM遍历完成后，reconciler才能给出当前需要修改真实DOM的信息，并传递给renderer，进行渲染，然后屏幕上才会显示此次更新内容

#### fiber reconciler

对于函数来说，这没什么问题，因为我们只想要函数的运行结果，但对于UI来说还需要考虑以下问题:

- 并不是所有的state更新都需要立即显示出来，比如屏幕之外的部分的更新

- 并不是所有的更新优先级都是一样的

- 对于某些高优先级的操作，应该是可以打断低优先级的操作执行的

fiber-reconciler下，操作是可以分成很多小部分，并且可以被中断的，fiber对象表征reconciliation阶段所能拆分的最小工作单元，整个结构是一个链表树。每个工作单元（fiber）执行完成后，都会查看是否还继续拥有主线程时间片，如果有继续下一个，如果没有则先处理其他高优先级事务，等主线程空闲下来继续执行

简单来说，每次只做一个很小的任务，做完后能够“喘口气儿”，回到主线程看下有没有什么更高优先级的任务需要处理，如果又则先处理更高优先级的任务，没有则继续执行

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