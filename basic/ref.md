#### 基本使用

##### Ref对象创建

ref对象长这样

```js
{
    current: null , // current指向ref对象获取到的实际内容，可以是dom元素，组件实例，或者其他。
}
```

###### 类组件创建ref

通过 React.createRef 创建一个 ref 对象

```js
class Index extends React.Component{
    constructor(props){
       super(props)
       this.currentDom = React.createRef(null)
    }
    componentDidMount(){
        console.log(this.currentDom)
    }
    render= () => <div ref={ this.currentDom } >ref对象模式获取元素或组件</div>
}
```

React.createRef底层比较简单，就是创建了一个对象，对象上的current属性用于保存通过 ref 获取的 DOM 元素，组件实例等

```js
export function createRef() {
  const refObject = {
    current: null,
  }
  return refObject;
}
```

###### 函数组件创建ref

通过useRef来创建一个ref对象

```js
export default function Index(){
    const currentDom = React.useRef(null)
    React.useEffect(()=>{
        console.log( currentDom.current ) // div
    },[])
    return  <div ref={ currentDom } >ref对象模式获取元素或组件</div>
}
```

useRef的底层逻辑和createRef差不多，createRef创建的ref对象可以直接绑定到组件实例上供后续使用，而函数组件每次更新都会重新执行，所以useRef不能像createRef一样直接把ref暴露出来，否则每次更新都会重新初始化ref，同时也解释了为什么不能在函数组件中使用createRef。实际上，useRef创建的ref对象挂到了函数组件对应的fiber上，函数组件每次执行，会从fiber上取ref对象

##### 获取ref

1. 字符串

ref是字符串的适合，如果是类组件，会把子组件的实例绑定在 this.refs 上；函数组件没有实例，不能被ref标记

```js
class Children extends Component{  
    render=()=><div>hello,world</div>
}
/* TODO:  Ref属性是一个字符串 */
export default class Index extends React.Component{
    componentDidMount(){
       console.log(this.refs)
    }
    render=()=> <div>
        <div ref="currentDom"  >字符串模式获取元素或组件</div>
        <Children ref="currentComInstance"  />
    </div>
}
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ca7efcd73fe429aa83bd91f068c5508~tplv-k3u1fbpfcp-watermark.awebp)

2. 函数

当用一个函数来标记 Ref 的时候，将作为 callback 形式，等到真实 DOM 创建阶段，执行 callback ，获取的 DOM 元素或组件实例，将以回调函数第一个参数形式传入

```js
class Children extends React.Component{  
    render=()=><div>hello,world</div>
}
/* TODO: Ref属性是一个函数 */
export default class Index extends React.Component{
    currentDom = null
    currentComponentInstance = null
    componentDidMount(){
        console.log(this.currentDom)
        console.log(this.currentComponentInstance)
    }
    render=()=> <div>
        <div ref={(node)=> this.currentDom = node }  >Ref模式获取元素或组件</div>
        <Children ref={(node) => this.currentComponentInstance = node  }  />
    </div>
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74ba71b6c4f5456eaf7cd46e51598fa4~tplv-k3u1fbpfcp-watermark.awebp)

3. ref对象

```js
class Children extends React.Component{  
    render=()=><div>hello,world</div>
}
export default class Index extends React.Component{
    currentDom = React.createRef(null)
    currentComponentInstance = React.createRef(null)
    componentDidMount(){
        console.log(this.currentDom)
        console.log(this.currentComponentInstance)
    }
    render=()=> <div>
         <div ref={ this.currentDom }  >Ref对象模式获取元素或组件</div>
        <Children ref={ this.currentComponentInstance }  />
   </div>
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/796e66d30ee84a62867fe264c5b5eca6~tplv-k3u1fbpfcp-watermark.awebp)

#### 高级用法

##### forwardRef 转发 Ref

##### 组件通信

##### 缓存数据

#### ref原理