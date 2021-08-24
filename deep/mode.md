react目前提供了三种模式

- Legacy Mode

目前的react17版本默认只暴露Legacy Mode模式的接口，其他模式还不稳定，会同步的进行Reconcile调和，也就是同步渲染，不能被打断，执行到底

```js
ReactDOM.render(<App />, rootNode)
```

- Blocking Mode

blocking模式是legacy模式到concurrent模式的一个过渡，渐进迁移，也可以说是Concurrent Mode的优雅降级模式，拥有小部分Concurrent Mode的特性，也是同步渲染

```js
ReactDOM.createBlockingRoot(rootNode).render(<App />)
```

- Concurrent Mode

react18版本将会推出，异步、并发的进行Reconcile调和，所谓并发，是指多个更新任务在同一个主线程上执行，更新任务可以在运行和暂停之间切换

```js
ReactDOM.createRoot(rootNode).render(<App />)
```

由于更新任务可能被中断，被标记微unstable的接口可能会执行两次，是真的不安全，要想使用Concurrent Mode必须完全兼容严格模式下的react

```js
import React from 'react';

function ExampleApplication() {
  return (
    <div>
      <Header />
      {/* 开启严格模式 */}
      <React.StrictMode>
        <div>
          <ComponentOne />
          <ComponentTwo />
        </div>
      </React.StrictMode>
      <Footer />
    </div>
  );
}
```

[严格模式](https://zh-hans.reactjs.org/docs/strict-mode.html)

参考
1. [Adopting Concurrent Mode](https://reactjs.org/docs/concurrent-mode-adoption.html#why-so-many-modes)