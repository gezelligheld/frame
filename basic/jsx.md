一段jsx代码如下

```js
<div>
   <TextComponent />
   <div>hello,world</div>
   let us learn React!
</div>
```

#### babel处理

经babel处理后，jsx元素节点被编译成react.createElement

```js
 React.createElement("div", null,
        React.createElement(TextComponent, null),
        React.createElement("div", null, "hello,world"),
        "let us learn React!"
    )
```

给个特写。React.createElement将jsx变成 element 对象，而React.cloneElement是以 element 元素为模板克隆并返回新的 React element 元素，返回元素的 props 是将新的 props 与原始元素的 props 浅层合并后的结果

```js
React.createElement(
  // 组建类型
  type,
  // 组件props
  [props],
  // 其他
  [...children]
)

const newReactElement =  React.cloneElement(reactElement,{} ,...newChildren )
```

#### react调和阶段

在调和阶段，React element 对象的每一个子节点都会形成一个与之对应的 fiber 对象，然后通过 sibling、return、child 将每一个 fiber 对象联系起来

React 针对不同 React element 对象会产生不同 tag (种类) 的fiber 对象

```js
export const FunctionComponent = 0;       // 函数组件
export const ClassComponent = 1;          // 类组件
export const IndeterminateComponent = 2;  // 初始化的时候不知道是函数组件还是类组件 
export const HostRoot = 3;                // Root Fiber 可以理解为跟元素 ， 通过reactDom.render()产生的根元素
export const HostPortal = 4;              // 对应  ReactDOM.createPortal 产生的 Portal 
export const HostComponent = 5;           // dom 元素 比如 <div>
export const HostText = 6;                // 文本节点
export const Fragment = 7;                // 对应 <React.Fragment> 
export const Mode = 8;                    // 对应 <React.StrictMode>   
export const ContextConsumer = 9;         // 对应 <Context.Consumer>
export const ContextProvider = 10;        // 对应 <Context.Provider>
export const ForwardRef = 11;             // 对应 React.ForwardRef
export const Profiler = 12;               // 对应 <Profiler/ >
export const SuspenseComponent = 13;      // 对应 <Suspense>
export const MemoComponent = 14;          // 对应 React.memo 返回的组件
```

最终jsx会变成如下的fiber结构

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/873f00b1255d4f5f8dac4954cf37dc9f~tplv-k3u1fbpfcp-watermark.image)