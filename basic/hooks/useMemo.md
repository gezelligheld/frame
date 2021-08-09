#### 使用

返回该值的 memoized 版本，该值仅在某个依赖项改变时才会更新

```js
const memoizedMemo = useMemo(() => {
    doSomething(a, b);
}, [a, b]);
```

> 函数组件中的变量尽量都用useMemo包裹

这里的依赖项是否变化使用Object.is来对比的，不适用于引用类型，需要借助@huse/previous-value库中的useOriginalCopy，防止多余的渲染

```js
import {usePreviousValue} from '@huse/previous-value';

const originContents = useOriginalCopy(contents);

const variable = useMemo(() => {
    // ...
}, [originContents]);
```

#### 原理

类似于shouldComponentUpdate，但是useMemo只是一个通用的无副作用的缓存Hook，并不会影响组件的渲染与否

```js
function useMemo(create, deps) {
    if (isMount) {
        return mountMemo(create, deps);
    }
    return updateMemo(create, deps);
}
function mountMemo(nextCreate,deps) {
    const hook = mountWorkInProgressHook();
    const nextDeps = deps === undefined ? null : deps;
    const nextValue = nextCreate();
    hook.memoizedState = [nextValue, nextDeps];
    return nextValue;
}

function updateMemo(nextCreate,deps){
    const hook = updateWorkInProgressHook();
    const nextDeps = deps === undefined ? null : deps;
    // 上一次的缓存结果
    const prevState = hook.memoizedState;
    if (prevState !== null) {
        if (nextDeps !== null) {
            const prevDeps = prevState[1];
            if (areHookInputsEqual(nextDeps, prevDeps)) {
                return prevState[0];
            }
        }
    }
    const nextValue = nextCreate();
    hook.memoizedState = [nextValue, nextDeps];
    return nextValue;
}
```

如果想完整实现shouldComponentUpdate的效果，可以使用React.memo，类似PureComponent

```js
const Button = React.memo((props) => {
  // 你的组件
});
```