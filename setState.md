#### setState之后发生了什么

调用setState函数之后，react对传入的参数与组件当前的状态合并，然后进行差异对比进行最小化的页面渲染

#### setState可能是异步的

假设所有setState都是同步的，每执行一次setState都会重新虚拟dom diff、dom修改等操作，性能很差

- setState只在合成事件和钩子函数中是异步的，react会作批量更新，将多个setState合并为一次更新

- setState在原生事件和setTimeout中都是同步的，且并不会批量更新

- setState本身执行的过程和代码都是同步的，只是合成事件和钩子函数在更新之前调用，导致在合成事件和钩子函数中没法立马拿到更新后的值，形成了所谓的“异步”

```js
constructor(){
    super()
    this.state={
        age:1
    }
}
componentDidMount(){
    this.setState({age:this.state.age+1})
    console.log(this.state.age); // 1
    this.setState({age:this.state.age+1})
    console.log(this.state.age); // 1
    setTimeout(()=>{
        this.setState({age:this.state.age+1})
        console.log(this.state.age); // 3
        this.setState({age:this.state.age+1})
        console.log(this.state.age); // 4
    })
}
```