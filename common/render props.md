render props将一个组件封装的状态或行为共享给其他需要相同状态的组件，是一个用于告知组件需要渲染什么内容的函数 prop

具有 render prop 的组件接受一个返回 React 元素的函数，并在组件内部通过调用此函数来实现自己的渲染逻辑

```js
<DataProvider render={data => (
  <h1>Hello {data.target}</h1>
)}/>
```

示例如下

```js
const RC = React.Component;

class Cat extends RC {
    render() {
        const mouse = this.props.mouse;
        return (
            <img src './cat.png' style={{position: 'absolute', left: mouse.x, top: mouse.y}} />
        );
    }
}

class Mouse extends RC {
    constructor(props) {
        super(props);
        this.state = {x: 0, y: 0};
    }

    handle = e => {
        this.setState({
            x: e.clientX,
            y: e.clientY
        });
    }

    render() {
        return (
            <div onMouseMove={this.handle}>
                {/* 动态决定要渲染的内容，相当于给父组件一个回调，同时将状态抛出，达到复用的效果 */}
                {this.props.render(this.state)}
            </div>
        );
    }
}

class MouseTracker extends RC {
    render() {
        return (
            <Mouse render={data => <Cat mouse={data} />} />
        );
    }
}
```

或者可以直接放到元素内部，Mouse组件中使用this.props.children获取

```js
<Mouse>
    {mouse => (
        <p>鼠标的位置是 {mouse.x}，{mouse.y}</p>
    )}
</Mouse>
```