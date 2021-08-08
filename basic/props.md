#### props是什么

在 React 应用中写的子组件，无论是函数组件 FunComponent ，还是类组件 ClassComponent ，父组件绑定在它们标签里的属性/方法，最终会变成 props 传递给它们。对于一些特殊的属性，比如说 ref 或者 key ，React 会在底层做一些额外的处理

props可以是

- 子组件渲染的数据源

- 通知父组件的回调函数

- 单纯的组件传递

- 渲染函数

- render props ， 将要渲染的内容放到了 children 属性上

- render component 插槽组件

标签内部的属性和方法会直接绑定在 props 对象的属性上，对于render props和render component会被绑定在 props 的 chidren 属性中

```js
/* children 组件 */
function ChidrenComponent(){
    return <div> In this chapter, let's learn about react props ! </div>
}
/* props 接受处理 */
class PropsComponent extends React.Component{
    componentDidMount(){
        console.log(this,'_this')
    }
    render(){
        const {  children , mes , renderName , say ,Component } = this.props
        const renderFunction = children[0]
        const renderComponent = children[1]
        /* 对于子组件，不同的props是怎么被处理 */
        return <div>
            { renderFunction() }
            { mes }
            { renderName() }
            { renderComponent }
            <Component />
            <button onClick={ () => say() } > change content </button>
        </div>
    }
}
/* props 定义绑定 */
class Index extends React.Component{
    state={  
        mes: "hello,React"
    }
    node = null
    say= () =>  this.setState({ mes:'let us learn React!' })
    render(){
        return <div>
            <PropsComponent  
               mes={this.state.mes}  // ① props 作为一个渲染数据源
               say={ this.say  }     // ② props 作为一个回调函数 callback
               Component={ ChidrenComponent } // ③ props 作为一个组件
               renderName={ ()=><div> my name is alien </div> } // ④ props 作为渲染函数
            >
                { ()=> <div>hello,world</div>  } { /* ⑤render props */ }
                <ChidrenComponent />             { /* ⑥render component */ }
            </PropsComponent>
        </div>
    }
}
```

#### props的作用

1. 在 React 组件层级 props 充当的角色

- 一方面父组件 props 可以把数据层传递给子组件去渲染消费。另一方面子组件可以通过 props 中的 callback ，来向父组件传递信息

- 可以将视图容器作为 props 进行渲染

2. 从 React 更新机制中 props 充当的角色

props 可以作为组件是否更新的重要准则，变化即更新，于是有了 PureComponent ，memo 等性能优化方案

#### 监听props的变化

- 类组件中

componentWillReceiveProps 可以作为监听props的生命周期，但是 React 已经不推荐使用 ，未来版本可能会被废弃，于是出现了这个生命周期的替代方案 getDerivedStateFromProps

- 函数组件中

用 useEffect 来作为 props 改变后的监听函数

```js
React.useEffect(()=>{
    // props 中number 改变，执行这个副作用。
    console.log('props改变：' ，props.number  )
},[ props.number ])
```

#### props chidren

- props 插槽组件

在 Container 组件中，通过 props.children 属性访问到 Chidren 组件，为 React element 对象

```js
<Container>
    <Children>
</Container>
```

可以根据需要控制 Chidren 是否渲染，比如鉴权；Container 可以用 React.cloneElement 强化 props (混入新的 props )，或者修改 Chidren 的子元素

- render props

可以将需要传给 Children 的 props 直接通过函数参数的方式传递给执行函数 children

```js
<Container>
   { (ContainerProps)=> <Children {...ContainerProps}  /> }
</Container>

function Container(props) {
    const ContainerProps = {
        name: 'alien',
        mes:'let us learn react'
    }
    return  props.children(ContainerProps)
}
```

如果children中既有react element也有函数，需要处理一下

```js
function  Container(props) {
    const ContainerProps = {
        name: 'alien',
        mes:'let us learn react'
    }
     return props.children.map(item=>{
        // 判断是 react elment  混入 props
        if(React.isValidElement(item)){
            return React.cloneElement(item, { ...ContainerProps }, item.props.children)
        }
        // 判断是 函数，直接执行
        else if(typeof item === 'function'){
            return item(ContainerProps)
        }
        return null
     })
}
```

#### 实现简易form

Form

```js
class Form extends React.Component{
    state={
        formData:{}
    }
    /* 用于提交表单数据 */
    submitForm=(cb)=>{
        cb({ ...this.state.formData })
    } 
    /* 获取重置表单数据 */
    resetForm=()=>{
       const { formData } = this.state
       Object.keys(formData).forEach(item=>{
           formData[item] = ''
       })
       this.setState({
           formData
       })
    }
    /* 设置表单数据层 */
    setValue=(name,value)=>{
        this.setState({
            formData:{
                ...this.state.formData,
                [name]:value
            }
        })
    }
    render(){
        const { children } = this.props
        const renderChildren = []
        React.Children.forEach(children,(child)=>{
            if(child.type.displayName === 'formItem'){
                const { name } = child.props
                /* 克隆`FormItem`节点，混入改变表单单元项的方法 */
                const Children = React.cloneElement(child,{ 
                    key:name ,                             /* 加入key */
                    handleChange: this.setValue ,           /* 用于改变 value */
                    value: this.state.formData[name] ||  '' /* value 值 */
                }, child.props.children)
                renderChildren.push(Children)
            }
        })
        return renderChildren
    }
}
/* 增加组件类型type  */
Form.displayName = 'form'
```

FormItem

```js
function FormItem(props){
    const { children , name  , handleChange , value , label  } = props
    const onChange = (value) => {
        /* 通知上一次value 已经改变 */
        handleChange(name,value)
    }
   return <div className='form' >
       <span className="label" >{ label }:</span>
       {
            React.isValidElement(children) && children.type.displayName === 'input' 
            ? React.cloneElement(children,{ onChange , value })
            : null
       }
   </div>    
}
FormItem.displayName = 'formItem'
```

Input

```js
function Input({ onChange , value }){
    return  <input className="input"  onChange={e => onChange && onChange(e.target.value)} value={value}  />
}
Input.displayName = 'input'
```

使用

```js
export default  () => {
    const form =  React.useRef(null)
    const submit =()=>{
        form.current.submitForm((formValue)=>{
            console.log(formValue)
        })
    }
    const reset = ()=>{
        form.current.resetForm()
    }
    return <div className='box' >
        <Form ref={ form } >
            <FormItem name="name" label="我是"  >
                <Input   />
            </FormItem>
            <FormItem name="mes" label="我想对大家说"  >
                <Input   />
            </FormItem>
            <input  placeholder="不需要的input" />
            <Input/>
        </Form>
        <div className="btns" >
            <button className="searchbtn"  onClick={ submit } >提交</button>
            <button className="concellbtn" onClick={ reset } >重置</button>
        </div>
    </div>
}
```