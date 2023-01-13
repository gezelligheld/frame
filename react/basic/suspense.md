#### 异步渲染

Suspense 让组件‘等待’异步操作，异步请求结束后在进行组件的渲染，fallback是Suspense处于loading状态下的显示内容

```js
// 当 UserInfo 处于数据加载状态下，展示 Suspense 中 fallback 的内容
function UserInfo() {
  const user = getUserInfo();
  return <h1>{user.name}</h1>;
}
function Index(){
    return <Suspense fallback={<h1>Loading...</h1>}>
        <UserInfo/>
    </Suspense>
}
```

#### 动态加载

配合lazy可以动态加载路由

```js
const LazyComponent = React.lazy(() => import('./test.js'))

function Index(){
   return <Suspense fallback={<div>loading...</div>} >
       <LazyComponent />
   </Suspense>
}
```

Suspense内部通过try catch捕获异常，抛出一个promise，进行数据请求工作，结束后会重新render将数据渲染出来。换言之，原本render是一个连续的过程，被Suspense包裹后，当发现有异步操作，会展示fallback中的内容，异步请求结束后再渲染数据

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60d20c4fad834541873697ead2ec6dda~tplv-k3u1fbpfcp-watermark.awebp)

lazy包裹的组件会被标记为REACT_LAZY_TYPE类型的element对象，在fiber调和阶段会变成LazyComponent类型的fiber节点

1. 首次渲染时执行init方法，执行lazy中传入的函数得到一个promise，此时payload._status并不是Resolved，抛出异常给到Suspense

2. Suspense捕获到该异常后，会执行该promise，得到最终要渲染的组件defaultExport，然后发起第二次渲染，此时init中的resolved._status已经是Resolved状态，直接返回要渲染的组件

```js
function lazy(ctor){
    return {
         $$typeof: REACT_LAZY_TYPE,
         _payload:{
            _status: -1,  //初始化状态
            _result: ctor,
         },
         _init:function(payload){
             if(payload._status===-1){ /* 第一次执行会走这里  */
                const ctor = payload._result;
                const thenable = ctor();
                payload._status = Pending;
                payload._result = thenable;
                thenable.then((moduleObject)=>{
                    const defaultExport = moduleObject.default;
                    resolved._status = Resolved; // 1 成功状态
                    resolved._result = defaultExport;/* defaultExport 为我们动态加载的组件本身  */ 
                })
             }
            if(payload._status === Resolved){ // 成功状态
                return payload._result;
            }
            else {  //第一次会抛出Promise异常给Suspense
                throw payload._result; 
            }
         }
    }
}
```