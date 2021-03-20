全局状态管理插件，利用context上下文和高阶组件进行封装

#### 使用

1. 在index.js中引入Provider作为根组件，并将store对象注册到根组件上

```js
import {Provider} from 'react-redux'
ReactDOM.render(
<Provider store={store}>
<App />
</Provider>
, document.getElementById('root'));
```

2. 在需要的组件中添加高阶组件connect，获取上下文

```js
import React from 'react'
import {connect} from 'react-redux'
import  Style from './index.module.less'
class Son1 extends React.Component{
  render(){
    return(
      <div className={Style.hehe}>
       son1{this.props.age}
      </div>
    )
  }
}
export default connect(state=>state)(Son1)
```

3. 高阶组件connect第一个括号内有两个参数

- mapStateToProps，接收state为参数，返回一个对象，映射到props中

```js
class Component extends React.Component{
  render(){
    console.log(this,'son1')
    return(
      <div>
      {/*直接通过props调用store全局状态值*/}
       son1{this.props.age}
      </div>
    )
  }
}
let mapStateToProps=(state)=>{
   return state
}
export default connect(mapStateToProps)(Component)
//可以简写为
export default connect(state=>state)(Component)
```

- mapDispatchToProps，接收dispatch为参数，返回一个对象，映射到props中

```js
import {bindActionCreators} from 'redux'
class Component extends React.Component{
  render(){
    console.log(this,'son1')
    return(
      <div>
      {/*直接通过props调用actionCreator中的方法*/}
       <button onClick={()=>{
        this.props.changeage()
       }}></button>
      </div>
    )
  }
}
let mapDispathToProps=(dispatch)=>{
   return bindActionCreators(ActionCreator,dispatch)
}
export default connect(state=>state,mapDispathToProps)(Component)
```

> 这样调用的话actionCreator中的方法直接向reducer抛发action即可

- 第二个参数mapDispatchToProps不传入时，通过props下的dispatch抛发action对象

```js
import actionCreator from './store/actionCreator.js'
class Component extends React.Component{
  render(){
    console.log(this,'son1')
    return(
      <div>
      {/*直接通过props调用dispatch方法抛发action对象*/}
       <button onClick={()=>{
        let action=actionCreator.changeage()
        this.props.dispatch(action)
       }}></button>
      </div>
    )
  }
}
export default connect(state=>state)(Component)
```

#### 原理

- Provider

将store对象挂到全局的上下文context下，实现如下

```js
class Provider extends React.Component {
    getChildContext() {
        return {store: this.store};
    }

    constructor(props, context) {
        super(props, context);
        this.store = props.store;
    }

    render() {
        return this.props.children;
    }
}
```

- connect

连接react组件和redux store，是个高阶组件，简易实现如下

```js
function connect(mapStateToProps, mapDispatchToProps, mergeProps, option = {}){
    return WrappedComponent => {
        class Connect extends React.Component {
            constructor(props, context) {
                super(props, context);
                this.store = props.store || context.store;
                this.state = {};
            }

            componentWillMount() {
                this.store.subscribe(() => {
                    this.setState(mapStateToProps(this.store.getState(), this.props));
                });
            }

            render() {
                return (
                    <WrappedComponent
                        {...this.state}
                        {...mapDispatchToProps(this.store.dispatch, props)}
                    />
                );
            }
        }

        return Connect;
    };
}
```