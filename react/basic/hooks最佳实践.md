探讨一下 hooks 代码如何更好的进行状态管理，提高代码可读性和可维护性

#### 尽量少的定义 state

1. 一些特殊的场景可以用 useRef

useRef 在一个生命周期内是不变的，要想获取最新的 ref 值，需要伴随着状态的变化；一些不需要引起 rerender 的值可以使用 ref 代替；全局变量可以用 ref 代替，一来不会影响全局命名空间，而且 ref 会随着组件的销毁而销毁

```jsx
function App() {
  const [, setForceUpdate] = useState(1);

  const ref = useRef(null);

  console.log(ref.current);

  useEffect(() => {
    const timer = setTimeout(() => {
      // 放到setstate前可以拿到更新后的值
      ref.current = Math.random();
      setForceUpdate(Math.random());
      // 这里是拿不到更新后的值的
      // ref.current = Math.random();
    }, 1000);

    return () => {
      clearTimeout(timer);
    };
  }, []);
}
```

2. 优先去派生状态

bad

```js
function Board() {
  const [squares, setSquares] = React.useState(Array(9).fill(null));
  const [nextValue, setNextValue] = React.useState(calculateNextValue(squares));
  const [winner, setWinner] = React.useState(calculateWinner(squares));
  const [status, setStatus] = React.useState(calculateStatus(squares));

  function selectSquare(square) {
    if (winner || squares[square]) {
      return;
    }
    const squaresCopy = [...squares];
    squaresCopy[square] = nextValue;
    const newNextValue = calculateNextValue(squaresCopy);
    const newWinner = calculateWinner(squaresCopy);
    const newStatus = calculateStatus(newWinner, squaresCopy, newNextValue);
    setSquares(squaresCopy);
    setNextValue(newNextValue);
    setWinner(newWinner);
    setStatus(newStatus);
  }

  // ...
}
```

good

```js
function Board() {
  const [squares, setSquares] = React.useState(Array(9).fill(null));

  const nextValue = useMemo(() => calculateNextValue(squares), [squares]);
  const winner = useMemo(() => calculateWinner(squares), [squares]);
  const status = useMemo(
    () => calculateStatus(winner, squares, nextValue),
    [winner, nextValue, squares]
  );

  function selectSquare(square) {
    if (winner || squares[square]) {
      return;
    }
    const squaresCopy = [...squares];
    squaresCopy[square] = nextValue;
    setSquares(squaresCopy);
  }
}
```

#### 善用 useContext

当一个状态需要下发到多个子组件时

```jsx
function Demo() {
  const [userList, setUserList] = useState([]);

  return (
    <div>
      <Child1 data={userList} />
      <Child2 data={userList} />
      ...
    </div>
  );
}
```

或者当一个状态需要层层传递时

```jsx
function Parent() {
    const [userList, setUserList] = useState([]);

    return (
        <div>
            <Child data={userList} />
        </div>
    );
}

function Child() {
    return <Child2 data={props.data}>
}
```

建议使用 useContext 进行优化，以第一个为例

```tsx
// context.ts
import { createContext } from 'react';

export default createContext<{ data: any[] }>({
  data: []
});

// index.tsx
import Context from './context';

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

然后在子组件中这样去获取

```tsx
import context from './context';

function Child1() {
  const { data } = useContext(context);
  // ...
}
```

这样可能有点麻烦，也可以用自定义 hook 去获取 context

```tsx
import { createContext, useContext } from 'react';

export const Context = createContext<{ a: string }>({ a: '' });

const useOverviewContext = () => useContext(Context);

export default useOverviewContext;
```

> 需要注意的时，Provider 的 value 发生变化后，所有被其包裹的消费组件都会 rerender，因此只需要将需要订阅 context 的组件包裹起来即可，防止多余的渲染

#### 善用 useReducer

进行状态的读写操作，与 useContext 结合使用可以达到一个局部的 redux 的效果

```jsx
const initState = {
  n: 0,
  m: 0,
  p: 0,
};

const reducer = (state, action) => {
  switch (action.type) {
    case 'setN':
      return { ...state, n: state.n + action.number };
    case 'setM':
      return { ...state, m: state.m + action.number };
    case 'setP':
      return { ...state, p: state.p + action.number };
    default:
      return state;
  }
};

const Context = React.createContext(null);
const App = () => {
  const [state, dispatch] = React.useReducer(reducer, initState);
  return (
    <Context.Provider value={{ state, dispatch }}>
      <N />
      <M />
      <P />
    </Context.Provider>
  );
};

const N = () => {
  const { state, dispatch } = React.useContext(Context);
  const addClick = () => {
    dispatch({ type: 'setN', number: 1 });
  };
  return (
    <div>
      <h1>N组件</h1>
      <div>n:{state.n}</div>
      <div>m:{state.m}</div>
      <div>p:{state.p}</div>
      <button onClick={addClick}>+1</button>
    </div>
  );
};

const M = () => {
  const { state, dispatch } = React.useContext(Context);
  const addClick = () => {
    dispatch({ type: 'setM', number: 2 });
  };
  return (
    <div>
      <h1>M组件</h1>
      <div>n:{state.n}</div>
      <div>m:{state.m}</div>
      <div>p:{state.p}</div>
      <button onClick={addClick}>+2</button>
    </div>
  );
};

const P = () => {
  const { state, dispatch } = React.useContext(Context);
  const addClick = () => {
    dispatch({ type: 'setP', number: 3 });
  };
  return (
    <div>
      <h1>P组件</h1>
      <div>n:{state.n}</div>
      <div>m:{state.m}</div>
      <div>p:{state.p}</div>
      <button onClick={addClick}>+3</button>
    </div>
  );
};
```

#### 状态分离和复用

借助自定义 hook、dva、mobx 等方式抽出 model 层，将视图和逻辑解耦。对于自定义 hook 而言，参考 ahooks 的处理方式，有以下两个比较好的规范

- 输出函数引用地址不变：借助 ahooks 中的 useMemoizedFn，源码如下

```js
function useMemoizedFn<T extends noop>(fn: T) {
  const fnRef = useRef<T>(fn);

  fnRef.current = useMemo(() => fn, [fn]);

  const memoizedFn = useRef<PickFunction<T>>();
  if (!memoizedFn.current) {
    memoizedFn.current = function (this, ...args) {
      return fnRef.current.apply(this, args);
    };
  }

  return memoizedFn.current as T;
}
```

- 输入函数调用时使用最新的引用：借助 ref 保存

```js
const useUnmount = (fn) => {
  const fnRef = useRef(fn);
  fnRef.current = fn;

  useEffect(
    () => () => {
      fnRef.current();
    },
    []
  );
};
```

参考

1. [Don't Sync State. Derive It!](https://kentcdodds.com/blog/dont-sync-state-derive-it)
