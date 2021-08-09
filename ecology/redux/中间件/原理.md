redux中间件提供的是action触发后，到达reducer之前的扩展，即将view -> action -> reducer -> store扩展为view -> action -> middleware -> reducer -> store，在这个环节下可以进行副作用操作，如异步请求、打印日志等

![middleware的作用](https://user-gold-cdn.xitu.io/2018/4/19/162dcb142c194bfb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

使用如下，以react-thunk为例

```js
import {createStore, compose, applyMiddleware} from 'redux';
import thunk from 'redux-thunk';

const store = createStore(rootReducer, applyMiddleware(thunk));
```

#### 原理

##### createStore

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

##### applyMiddleware

本质上是对dispatch做了改造，实现如下

```js
function applyMiddleware(...middlewares) {
    return createStore => {
        return reducer => {
            const store = createStore(reducer);
            let dispatch = store.dispatch;
            // 每个middleware接受这两个参数
            const middlewareParam = {
                getState: store.getState,
                dispatch: action => dispatch(action)
            };
            // 可以传入多个middleware，依次执行，且无需关心前后的middleware
            // middleware的形式({ getState, dispatch }) => next => action
            // chain的形式 Array<next => action>
            const chain = middlewares.map(middleware => middleware(middlewareParam));
            // compose(..chain)返回一个middleware，形式next => action，然后执行，这里的next相当于store.dispatch
            // 此时的dispatch的形式是 action => any
            dispatch = compose(...chain)(store.dispatch);
            // 覆盖掉原来的dispatch
            return {...store, dispatch};
        };
    };
}
```

##### compose

```js
function add1(str){
   return str+1;
}
function add2(str){
    return str+2;
}
function add3(str){
    return str+3;
}
let result = add3(add2(add1('好吃'))); // 好吃123
```

这样层层嵌套显然不太合适，类比传入了多个middleware，实现如下

```js
function compose(...fns) {
    // fns的形式是 Array<next => action>
    if (fns.length === 1) {
        return fns[0];
    }
    // 这里的a、b是前后两个中间件
    // 后一个middleware会以前一个middleware的结果入参数，从右向左依次执行
    return fns.reduce((a, b) => (...args) => a(b(...args)));
}

const add = compose(add3, add2, add1);
const result = add('好吃'); // '好吃123'
```

##### middleware

以react-thunk为例，用来解决异步dispatch action对象的问题

```js
const todo = () => {
    return async dispatch => {
        const data = await request();
        dispatch({
            type: 'todo',
            data
        });
    }
};
```

实现如下，同样是({ getState, dispatch }) => next => action的形式

```js
function thunk({getState, dispatch}) {
    // next是applyMiddleware透传下来的store.dispatch，是初始时没有被改造过的dispatch
    return next => {
        // actions是actionCreators下的发起action的函数，含有异步操作时返回了函数
        return action => {
            if (typeof action === 'function') {
                return action(dispatch, getState);
            }
            // action不是函数则不处理
            return next(action);
        };
    };
}
```

综上，applyMiddleware和thunk将dispatch改造后，react组件调用store.dispatch时，相当于执行了发起action的函数，如果存在异步操作时，待异步操作完成后再使用没有被改造过的dispatch去通知store修改数据

参考文档
1. [redux中间件](https://juejin.cn/post/6844903593749774344#heading-4)
2. [对于react-thunk中间件的简单理解](https://blog.csdn.net/weixin_38642331/article/details/81748312)