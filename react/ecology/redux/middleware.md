redux 中间件提供的是 action 触发后，到达 reducer 之前的扩展，即将 view -> action -> reducer -> store 扩展为 view -> action -> middleware -> reducer -> store，在这个环节下可以进行副作用操作，如异步请求、打印日志等

![middleware的作用](https://user-gold-cdn.xitu.io/2018/4/19/162dcb142c194bfb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

使用如下，以 react-thunk 为例

```js
import { createStore, compose, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';

const store = createStore(rootReducer, applyMiddleware(thunk));
```

这样就允许我们的 action 作为一个函数来发送异步请求了

```js
const fetchList = () => {
  return async (dispatch) => {
    const list = await api.getList();
    dispatch({
      type: FETCH_LIST,
      payload: {
        list,
      },
    });
  };
};
dispatch(fetchList());
```

#### 原理

createStore 通过 enhancer 返回一个增强的 store

```js
// 第二个参数preloadedState一般作为整个应用的初始化数据，如果传入了这个参数，applyMiddleware就会被当做第三个参数处理
function createStore(reducer, preloadedState, enhancer) {
  if (typeof preloadedState === 'function' && enhancer === undefined) {
    enhancer = preloadedState;
    preloadedState = undefined;
  }

  if (enhancer !== undefined) {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function');
    }
    return enhancer(createStore)(reducer, preloadedState);
  }
  // ...
}
```

enhancer 接收一个 createStore 参数，返回一个增强的 store，一般结构如下

```js
const enhancer = () => {
  return (createStore) => (reducer, initState, enhancer) => {
    // ...
    return {
      ...store,
      dispatch,
    };
  };
};
```

applyMiddleware 是一个典型的 enhancer，核心在于用一个 compose 函数将多个中间件串连起来

```js
function applyMiddleware(...middlewares) {
  return (createStore) => {
    return (reducer) => {
      const store = createStore(reducer);
      let dispatch = store.dispatch;
      // 每个middleware接受这两个参数
      const middlewareParam = {
        getState: store.getState,
        dispatch: (action) => dispatch(action),
      };
      // 可以传入多个middleware，依次执行，且无需关心前后的middleware
      // middleware的形式({ getState, dispatch }) => next => action
      // chain的形式 Array<next => action>
      const chain = middlewares.map((middleware) =>
        middleware(middlewareParam)
      );
      // compose(..chain)返回一个middleware，形式next => action，然后执行，这里的next相当于store.dispatch
      // 此时的dispatch的形式是 action => any
      dispatch = compose(...chain)(store.dispatch);
      // 覆盖掉原来的dispatch
      return { ...store, dispatch };
    };
  };
}
```

compose 组合是函数式编程中的概念，可以创建一个从右向左的数据流

```js
// 初始
function add1(str) {
  return str + 1;
}
function add2(str) {
  return str + 2;
}
function add3(str) {
  return str + 3;
}
let result = add3(add2(add1('好吃'))); // 好吃123

// compose
function compose(...fns) {
  // fns的形式是 Array<next => action>
  if (fns.length === 1) {
    return fns[0];
  }
  // 这里的a、b是前后两个中间件
  // 后一个middleware会以前一个middleware的结果入参数，从右向左依次执行
  return fns.reduce(
    (a, b) =>
      (...args) =>
        a(b(...args))
  );
}

const add = compose(add3, add2, add1);
const result = add('好吃'); // '好吃123'
```

#### 中间件

中间件的实现原理也很简单，可以理解为在 action 和 reducer 之间对 dispatch 做了一次增强

##### react-thunk

以 react-thunk 为例，用来解决异步 dispatch action 对象的问题

```js
const todo = () => {
  return async (dispatch) => {
    const data = await request();
    dispatch({
      type: 'todo',
      data,
    });
  };
};
```

实现如下，同样是({ getState, dispatch }) => next => action 的形式

```js
function thunk({ getState, dispatch }) {
  // next是applyMiddleware透传下来的store.dispatch，是初始时没有被改造过的dispatch
  return (next) => {
    // actions是actionCreators下的发起action的函数，含有异步操作时返回了函数
    return (action) => {
      if (typeof action === 'function') {
        return action(dispatch, getState);
      }
      // action不是函数则不处理
      return next(action);
    };
  };
}
```

applyMiddleware 和 thunk 将 dispatch 改造后，react 组件调用 store.dispatch 时，相当于执行了发起 action 的函数，如果存在异步操作时，待异步操作完成后再使用没有被改造过的 dispatch 去通知 store 修改数据

##### logger

```js
const logger = (middlewareAPI) => {
  return (next) => {
    return (action) => {
      console.log('dispatch 前：', middlewareAPI.getState());
      var returnValue = next(action);
      console.log('dispatch 后：', middlewareAPI.getState(), '\n');
      return returnValue;
    };
  };
};
```

参考文档

1. [redux 中间件](https://juejin.cn/post/6844903593749774344#heading-4)
2. [对于 react-thunk 中间件的简单理解](https://blog.csdn.net/weixin_38642331/article/details/81748312)
