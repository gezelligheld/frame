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

ref是字符串的适合，如果是类组件，会把子组件的实例绑定在 this.refs 上；函数组件没有实例，不能被ref标记。但这种获取ref的方式被标记为了unstable，concurrent mode下不支持，少用

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

forwardRef 的初衷就是解决 ref 不能跨层级捕获和传递的问题，它接受父组件标记的ref信息并将它转发下去，子组件可以通过props获取上一层级或者更高层级的ref，然后可以向ref对象中注入数据、组件实例等

1. 跨层级获取

如下，GrandFather通过标记ref获取到了Son组件中的某个元素

```js
function Son (props){
    const { grandRef } = props
    return <div>
        <div> i am alien </div>
        <span ref={grandRef} >这个是想要获取元素</span>
    </div>
}

class Father extends React.Component{
    constructor(props){
        super(props)
    }
    render(){
        return <div>
            <Son grandRef={this.props.grandRef}  />
        </div>
    }
}

// 将父组件标记的ref转发下去
const NewFather = React.forwardRef((props,ref)=> <Father grandRef={ref}  {...props} />)

class GrandFather extends React.Component{
    constructor(props){
        super(props)
    }
    node = null 
    componentDidMount(){
        console.log(this.node) // span #text 这个是想要获取元素
    }
    render(){
        return <div>
            <NewFather ref={(node)=> this.node = node } />
        </div>
    }
}
```

2. 合并转发ref

```js
class Form extends React.Component{
    render(){
       return <div>{...}</div>
    }
}

class Index extends React.Component{ 
    componentDidMount(){
        const { forwardRef } = this.props
        forwardRef.current={
            form:this.form,
            index:this,
            button:this.button,
        }
    }
    form = null
    button = null
    render(){
        return <div   > 
          <button ref={(button)=> this.button = button }  >点击</button>
          <Form  ref={(form) => this.form = form }  />  
      </div>
    }
}

// 将父组件标记的ref转发下去
const ForwardRefIndex = React.forwardRef(( props,ref )=><Index  {...props} forwardRef={ref}  />)

function Home(){
    const ref = useRef(null)
     useEffect(()=>{
         console.log(ref.current)
     },[])
    return <ForwardRefIndex ref={ref} />
}
```

3. 高阶组件转发

高阶组件包裹一个组件，返回一个新组件，如果没有对ref处理，那么ref会指向高阶组件返回的新组件，而不是包裹的原组件，所以需要转发一下

```js
function HOC(Component){
  class Wrap extends React.Component{
     render(){
        const { forwardedRef ,...otherprops  } = this.props
        return <Component ref={forwardedRef}  {...otherprops}  />
     }
  }
  return  React.forwardRef((props,ref)=> <Wrap forwardedRef={ref} {...props} /> ) 
}

class Index extends React.Component{
  render(){
    return <div>hello,world</div>
  }
}

const HocIndex =  HOC(Index)

export default ()=>{
  const node = useRef(null)
  useEffect(()=>{
    console.log(node.current)  /* Index 组件实例  */ 
  },[])
  return <div><HocIndex ref={node}  /></div>
}
```

##### 组件通信

1. 类组件ref

类组件可以通过 ref 直接获取组件实例，实现组件通信

```js
class Son extends React.PureComponent{
    state={
       fatherMes:'',
       sonMes:''
    }
    fatherSay=(fatherMes)=> this.setState({ fatherMes  }) /* 提供给父组件的API */
    render(){
        const { fatherMes, sonMes } = this.state
        return <div className="sonbox" >
            <div className="title" >子组件</div>
            <p>父组件对我说：{ fatherMes }</p>
            <div className="label" >对父组件说</div> <input  onChange={(e)=>this.setState({ sonMes:e.target.value })}   className="input"  /> 
            <button className="searchbtn" onClick={ ()=> this.props.toFather(sonMes) }  >to father</button>
        </div>
    }
}

function Father(){
    const [ sonMes , setSonMes ] = React.useState('') 
    const sonInstance = React.useRef(null) /* 用来获取子组件ref对象上挂的实例 */
    const [ fatherMes , setFatherMes ] = React.useState('')
    const toSon = () => sonInstance.current.fatherSay(fatherMes) /* 调用子组件实例方法，改变子组件state */
    return <div className="box" >
        <div className="title" >父组件</div>
        <p>子组件对我说：{ sonMes }</p>
        <div className="label" >对子组件说</div> <input onChange={ (e) => setFatherMes(e.target.value) }  className="input"  /> 
        <button className="searchbtn"  onClick={toSon}  >to son</button>
        <Son ref={sonInstance} toFather={setSonMes} />
    </div>
}
```

2. 函数组件 forwardRef + useImperativeHandle

useImperativeHandle接受forwardRef转发的父组件标识的ref，将第二个参数的返回值注入到ref中，供父组件获取子组件内的一些数据

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59238390306849e89069e6a4bb6ded9d~tplv-k3u1fbpfcp-watermark.awebp)

```js
function Son (props,ref) {
    const inputRef = useRef(null)
    const [ inputValue , setInputValue ] = useState('')
    useImperativeHandle(ref,()=>{
       const handleRefs = {
           onFocus(){              /* 声明方法用于聚焦input框 */
              inputRef.current.focus()
           },
           onChangeValue(value){   /* 声明方法用于改变input的值 */
               setInputValue(value)
           }
       }
       return handleRefs
    },[])
    return <div>
        <input placeholder="请输入内容"  ref={inputRef}  value={inputValue} />
    </div>
}

const ForwarSon = forwardRef(Son)

class Index extends React.Component{
    cur = null
    handerClick(){
       const { onFocus , onChangeValue } = this.cur
       onFocus() // 让子组件的输入框获取焦点
       onChangeValue('let us learn React!') // 让子组件input  
    }
    render(){
        return <div style={{ marginTop:'50px' }} >
            <ForwarSon ref={cur => (this.cur = cur)} />
            <button onClick={this.handerClick.bind(this)} >操控子组件</button>
        </div>
    }
}
```

##### 函数组件缓存数据

函数组件每一次更新，都会重新执行函数上下文，可以把一些不依赖于视图更新的数据储存到 ref 对象中，只要组件没有销毁，ref 对象就一直存在

存到ref里主要有两个好处

- 能够直接修改数据，不会造成函数组件冗余的更新

- 无须将 ref 对象添加成 dep 依赖项，因为 useRef 始终指向一个内存空间，所以这样一点好处是可以随时访问到变化后的值

```js
const toLearn = [ { type: 1 , mes:'let us learn React' } , { type:2,mes:'let us learn Vue3.0' }  ]
export default function Index({ id }){
    const typeInfo = React.useRef(toLearn[0])
    const changeType = (info)=>{
        typeInfo.current = info
    }
    useEffect(()=>{
       if(typeInfo.current.type===1){
           /* ... */
       }
    },[ id ]) /* 无须将 typeInfo 添加依赖项  */
    return <div>
        {
            toLearn.map(item=> <button key={item.type}  onClick={ changeType.bind(null,item) } >{ item.mes }</button> )
        }
    </div>
}
```

#### ref原理

如下代码，当点击目标元素时，第一次输出null，第二次才输出div元素

```js
export default class Index extends React.Component{
    state={ num:0 }
    node = null
    render(){
        return <div >
            <div ref={(node)=>{
               this.node = node
               console.log('此时的参数是什么：', this.node )
            }}  >ref元素节点</div>
            <button onClick={()=> this.setState({ num: this.state.num + 1  }) } >点击</button>
        </div>
    }
}
```

为什么会这样？组件更新分为render阶段和commit阶段，对于ref的处理是在commit阶段发生的

首先会在在DOM更新之前，也就是组件更新的commit阶段的mutation阶段，执行commitDetachRef，commitDetachRef会清空之前的ref，置为null，方便垃圾回收

```js
function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === 'function') { /* function 和 字符串获取方式。 */
      currentRef(null); 
    } else {   /* Ref对象获取方式 */
      currentRef.current = null;
    }
  }
}
```

DOM更新之后，需要更新ref，执行commitAttachRef

```js
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent: //元素节点 获取元素
        instanceToUse = getPublicInstance(instance);
        break;
      default:  // 类组件直接使用实例
        instanceToUse = instance;
    }
    if (typeof ref === 'function') {
      ref(instanceToUse);  //* function 和 字符串获取方式。 */
    } else {
      ref.current = instanceToUse; /* function 和 字符串获取方式。 */
    }
  }
}
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08a2393077634beaad2b91f971ab381f~tplv-k3u1fbpfcp-watermark.awebp)