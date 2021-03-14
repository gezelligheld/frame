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