concurrent是一种渲染模式，能使 React 在长时间渲染的场景下依旧保持良好的交互性，能优先执行高优先级变更，不会使页面处于卡顿或无响应状态，从而提升应用的用户体验

#### requestIdleCallback

JS 是单线程的，每当 JS 线程执行时，UI 渲染线程会挂起，UI 更新会被保存在队列中，等待 JS 线程空闲后立即被执行，事件触发也是如此；因此，长时间的 JS 持续执行，就会造成 UI 渲染线程长时间地挂起，触发的事件也得不到响应，用户层面就会感知到页面卡顿甚至卡死了

在一帧中，我们需要将 JS 执行时间控制在合理的范围内，不影响后续 Layout 与 Paint 的过程。requestIdleCallback 就能够充分利用帧与帧之间的空闲时间来执行 JS，可以根据 callback 传入的 dealine 判断当前是否还有空闲时间用于执行。由于浏览器可能始终处于繁忙的状态，导致 callback 一直无法执行，它还能够设置超时时间，一旦超过时间能使任务被强制执行

由于浏览器自带的requestIdleCallback在layout和paint后触发，如果callback中涉及到了DOM操作，又会重新触发layout和paint，再加上兼容性比较差，react内部模拟实现了requestIdleCallback，requestAnimationFrame 作为 ployfill，通过 帧率动态调整，计算空闲时间，而且requestAnimationFrame在layout和paint之前触发，方便进行DOM操作

> requestAnimationFrame方法会在每一帧被调用，可以把js分成小的任务块分配到每一帧，在每一帧用完前暂停js的执行，归还主线程，这样在下一帧开始时主线程可以按时开始布局和绘制

#### 任务按时间片拆分及时间片间的中断与恢复

Fiber树的更新流程分为render和commit阶段，在 Concurrent 模式下，render 阶段可以被拆解，每个时间片内分别运行一部分，直至完成；commit阶段会进行DOM更新操作，所以需要一次性更新完成

render阶段遍历Fiber树时，每遍历一次都会进行时间片的检查，如果时间片到了跳出循环，相当于render阶段被中断，当前处理的节点会被保留，等待下一个时间分片到来时继续处理

![image](https://pic3.zhimg.com/80/v2-7051c10497a34d2e65e5f4caabadf152_1440w.jpg)

> 这里的时间分片指一个16.67ms的渲染帧

#### 任务优先级划分

任务优先级通过过期时间（expiration time）界定

- 如果高优任务一直执行，低优任务便无法执行，给低优任务设置过期时间，一旦超时便会当作同步任务处理，立刻执行

- 表示更新优先级，expiration time越大，优先级越高

> concurrent下有时间线的概念，当前时间一方面与时间片的截止时间比较，来决定是否中断当前js的执行；另一方面与Fiber的过期时间比较，来决定是否继续向下遍历

当Fiber节点变更时，将带有过期时间的update对象插入到Fiber节点的updateQueue中，同一队列中如果存在低优任务和高优任务，会跳过低优任务；然后记录首个跳过的低优update对象，等待高优任务完成后以此为节点开始执行之后的任务

参考文档

1. [react concurrent](https://zhuanlan.zhihu.com/p/60307571?utm_source=tuicool)