全局状态管理插件，利用 context 上下文和高阶组件进行封装

#### 使用

1. 在 index.js 中引入 Provider 作为根组件，并将 store 对象注册到根组件上

```js
import { Provider } from 'react-redux';
ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
);
```

2. 在需要的组件中添加高阶组件 connect，获取上下文

```js
import React from 'react';
import { connect } from 'react-redux';
import Style from './index.module.less';
class Son1 extends React.Component {
  render() {
    return <div className={Style.hehe}>son1{this.props.age}</div>;
  }
}
export default connect((state) => state)(Son1);
```

3. 高阶组件 connect 第一个括号内有两个参数

- mapStateToProps，接收 state 为参数，返回一个对象，映射到 props 中

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

- mapDispatchToProps，接收 dispatch 为参数，返回一个对象，映射到 props 中

```js
import { bindActionCreators } from 'redux';
class Component extends React.Component {
  render() {
    console.log(this, 'son1');
    return (
      <div>
        {/*直接通过props调用actionCreator中的方法*/}
        <button
          onClick={() => {
            this.props.changeage();
          }}
        ></button>
      </div>
    );
  }
}
let mapDispathToProps = (dispatch) => {
  return bindActionCreators(ActionCreator, dispatch);
};
export default connect((state) => state, mapDispathToProps)(Component);
```

> 这样调用的话 actionCreator 中的方法直接向 reducer 抛发 action 即可

- 第二个参数 mapDispatchToProps 不传入时，通过 props 下的 dispatch 抛发 action 对象

```js
import actionCreator from './store/actionCreator.js';
class Component extends React.Component {
  render() {
    console.log(this, 'son1');
    return (
      <div>
        {/*直接通过props调用dispatch方法抛发action对象*/}
        <button
          onClick={() => {
            let action = actionCreator.changeage();
            this.props.dispatch(action);
          }}
        ></button>
      </div>
    );
  }
}
export default connect((state) => state)(Component);
```

#### 原理

- Provider

将 store 对象挂到全局的上下文 context 下，实现如下

```js
class Provider extends React.Component {
  getChildContext() {
    return { store: this.store };
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

连接 react 组件和 redux store，是个高阶组件，简易实现如下

```js
const connect = (mapStateToProps, mapDispathToProps) => (WrappedComponent) => {
  return class extends React.Component {
    static contextType = ReactReduxContext;
    constructor(props) {
      super(props);
      this.store = this.context.store;
      this.state = {
        state: this.store.getState(),
      };
    }
    componentDidMount() {
      this.store.subscribe((nextState) => {
        // 浅比较
        if (!shadowCompare(nextState, this.state.state)) {
          this.setState({ state: nextState });
        }
      });
    }
    render() {
      const props = {
        ...mapStateToProps(this.state.state),
        ...mapDispathToProps(this.state.state),
        ...this.props,
      };
      return <WrappedComponent {...props} />;
    }
  };
};
```
