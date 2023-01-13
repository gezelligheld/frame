Context 提供了一个无需为每层组件手动添加 props，就能在组件树间进行数据传递的方法

#### 基本使用

createContext可以创建一个context上下文对象

```js
const ThemeContext = React.createContext(null) //
const ThemeProvider = ThemeContext.Provider  //提供者
const ThemeConsumer = ThemeContext.Consumer // 订阅消费者
```

Provider的value属性可以传递context数据供消费者使用，当value改变时，消费该数据的组件会重新渲染

获取上下文的数据，有以下三种方式

1. contextType，只适用于类组件

```js
const ThemeContext = React.createContext(null)
// 类组件 - contextType 方式
class ConsumerDemo extends React.Component{
   render(){
       const { color,background } = this.context
       return <div style={{ color,background } } >消费者</div> 
   }
}
ConsumerDemo.contextType = ThemeContext

const Son = ()=> <ConsumerDemo />
```

2. useContext，只适用于函数组件

```js
const ThemeContext = React.createContext(null)
// 函数组件 - useContext方式
function ConsumerDemo(){
    const  contextValue = React.useContext(ThemeContext) /*  */
    const { color,background } = contextValue
    return <div style={{ color,background } } >消费者</div> 
}
const Son = ()=> <ConsumerDemo />
```

3. Consumer

采取 render props 方式，接受最近一层 provider 中value 属性，作为 render props 函数的参数，可以将参数取出来，作为 props 混入 ConsumerDemo 组件

```js
const ThemeConsumer = ThemeContext.Consumer // 订阅消费者

function ConsumerDemo(props){
    const { color,background } = props
    return <div style={{ color,background } } >消费者</div> 
}
const Son = () => (
    <ThemeConsumer>
       { /* 将 context 内容转化成 props  */ }
       { (contextValue)=> <ConsumerDemo  {...contextValue}  /> }
    </ThemeConsumer>
)
```

需要注意的是，当value改变时，Provider所包裹的组件默认都会重新渲染一遍，会造成不必要的渲染，可以通过memo、pureComponent作浅比较处理，或利用useMemo、useCallback等做缓存，减少重新渲染带来的额外的计算

#### 其他用法

##### 嵌套 Provider

```js
const ThemeContext = React.createContext(null) // 主题颜色Context
const LanContext = React.createContext(null) // 主题语言Context

function ConsumerDemo(){
    return <ThemeContext.Consumer>
        { (themeContextValue)=> (
            <LanContext.Consumer>
                { (lanContextValue) => {
                    const { color , background } = themeContextValue
                    return <div style={{ color,background } } >
                      { lanContextValue === 'CH'  ? '大家好，让我们一起学习React!' : 'Hello, let us learn React!'  }
                    </div> 
                } }
            </LanContext.Consumer>
        )  }
    </ThemeContext.Consumer>
}

const Son = memo(() => <ConsumerDemo />);

export default function ProviderDemo(){
    const [ themeContextValue ] = React.useState({  color:'#FFF', background:'blue' })
    const [ lanContextValue ] = React.useState('CH') // CH -> 中文 ， EN -> 英文
    return <ThemeContext.Provider value={themeContextValue}  >
         <LanContext.Provider value={lanContextValue} >
             <Son  />
         </LanContext.Provider>
    </ThemeContext.Provider>
}
```

##### 逐层传递Provider

一个context可以被多个Provider传递，下一层级的Provider会覆盖上一层级的Provider

```js
// 逐层传递Provder
const ThemeContext = React.createContext(null)
function Son2(){
    return <ThemeContext.Consumer>
        { (themeContextValue2)=>{
            const { color , background } = themeContextValue2
            return  <div  className="sonbox"  style={{ color,background } } >  第二层Provder </div>
        }  }
    </ThemeContext.Consumer>
}
function Son(){
    const { color, background } = React.useContext(ThemeContext)
    const [ themeContextValue2 ] = React.useState({  color:'#fff', background:'blue' }) 
    /* 第二层 Provder 传递内容 */
    return <div className='box' style={{ color,background } } >
        第一层Provder
        <ThemeContext.Provider value={ themeContextValue2 } >
            <Son2  />
        </ThemeContext.Provider>
    </div>

}

export default function Provider1Demo(){
    const [ themeContextValue ] = React.useState({  color:'orange', background:'pink' })
     /* 第一层  Provider 传递内容  */
    return <ThemeContext.Provider value={ themeContextValue } >
        <Son/>
    </ThemeContext.Provider> 
}
```