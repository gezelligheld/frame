#### 概念

Transition 用于一些不是很急迫的更新，需要在 concurrent 模式下使用

现在有这样一个场景，有一个 input 表单。并且有一个大量数据的列表，通过表单输入内容，对列表数据进行搜索，这就需要页面上 input 输入内容和列表数据进行变化，正常情况下我们期望 input 输入内容的更新实时显示，优先级要更高，就可以使用 startTransition 将列表数据的更新低优处理

```js
export default function App() {
  const [value, setInputValue] = React.useState('');
  const [isTransition, setTransion] = React.useState(false);
  const [query, setSearchQuery] = React.useState('');
  const handleChange = (e) => {
    setInputValue(e.target.value);
    if (isTransition) {
      React.startTransition(() => {
        setSearchQuery(e.target.value);
      });
    } else {
      setSearchQuery(e.target.value);
    }
  };
  return (
    <div>
      <button onClick={() => setTransion(!isTransition)}>
        {isTransition ? 'transition' : 'normal'}{' '}
      </button>
      <input onChange={handleChange} placeholder="输入搜索内容" value={value} />
      <NewList query={query} />
    </div>
  );
}
```

相比于用 setTimeout 实现，setTimeout 的回调是异步的，而 startTransition 是同步的，要比 setTimeout 的回调更早执行；相比于用防抖实现，但本质上也是用 setTimeout 实现

#### 特性

startTransition 接收一个回调函数作为参数，里面的更新任务都会被标记成过渡更新任务，过渡更新任务在渲染并发场景下，会被降级更新优先级，中断更新

过渡更新任务在未被执行前处于 pending 状态，useTransition 可以判断是否处于过渡状态，相当于 useState + startTransition

```js
import { useTransition } from 'react';

const [isPending, startTransition] = useTransition();
```

#### useDeferredValue

让状态滞后派生，相当于 useEffect + transtion，本质上和内部实现与 useTransition 一样都是标记成了过渡更新任务

```js
export default function App() {
  const [value, setInputValue] = React.useState('');
  const query = React.useDeferredValue(value);
  const handleChange = (e) => {
    setInputValue(e.target.value);
  };
  return (
    <div>
      <button>useDeferredValue</button>
      <input onChange={handleChange} placeholder="输入搜索内容" value={value} />
      <NewList query={query} />
    </div>
  );
}
```

参考

1. [React18 新特性深入浅出用户体验大师—transition](https://juejin.cn/post/7027995169211285512)
