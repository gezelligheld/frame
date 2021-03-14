#### 使用

- 给函数组件增加了操作副作用的能力，它跟class组件中的componentDidMount、componentDidUpdate和componentWillUnmount具有相同的用途，只不过被合并成了一个 API

```js
import React,{useState,useEffect} from 'react'

function userState(){
    useEffect(()=>{
        //会在组件挂在时和组件更新时调用，即componentDidMount和componentDidUpdate
        //第二个参数是一个数组，存放该hook所依赖的state和函数，只有相关依赖改变时才触发componentDidUpdate重新渲染页面
        //当第二个参数为空数组或不写时，不会触发componentDidUpdate
        return ()=>{
            //组件卸载时触发该回调，相当于componentWillUnmount
        }
    },[name])
    return (
        <div></div>
    )
}

export default userState
```

- 可以将不同的业务逻辑分离到多个useEffect中，顺序执行

#### 原理