浏览器的渲染线程和js线程是互斥的，当执行js的时候，渲染线程会挂起，时间长了就会造成页面卡顿的感觉。16版本之前的react，如果状态发生更新，会从根节点开始进行diff，递归遍历所有的虚拟DOM找到不同，然后更新DOM，这样同步更新的方式可能会造成卡顿

对于函数来说，这没什么问题，因为我们只想要函数的运行结果，但对于UI来说还需要考虑以下问题:

- 并不是所有的state更新都需要立即显示出来，比如屏幕之外的部分的更新

- 并不是所有的更新优先级都是一样的

- 对于某些高优先级的操作，应该是可以打断低优先级的操作执行的

16版本后采用了新的调度方式，react实现了一套自己的调度方式，更新任务可以分成很多小部分，甚至可以中断，以一个fiber作为最小的工作单元执行完毕后，都会去查看是否还继续拥有主线程时间片，如果有继续执行下一个fiber的更新任务，没有则进行其他高优的任务，等主线程进入空闲时间后再继续

Scheduler作为react中一个独立的包，承担着任务调度的职责，只需要将任务优先级和任务提交给它，就可以帮助管理任务

#### 时间分片

浏览器每次执行一次事件循环（一帧）都会做如下事情：处理事件，执行 js ，调用 requestAnimation ，布局 Layout ，绘制 Paint ，在一帧执行后，如果没有其他事件，那么浏览器会进入空闲时间，那么有的一些不是特别紧急 React 更新，就可以执行了

时间分片规定的是单个任务在这一帧内最大的执行时间，任务一旦执行时间超过时间片，则会被打断，有节制地执行任务。这样可以保证页面不会因为任务连续执行的时间过长而产生卡顿

浏览器提供的requestIdleCallback，当进入空闲时间后，触发其回调

```js
// timeout 超时时间，如果长时间没有空闲时间回调不会执行，超过超时时间会强制执行
requestIdleCallback(callback,{ timeout })
```

为了防止 requestIdleCallback 中的任务由于浏览器没有空闲时间而卡死，所以设置了 5 个优先级

- Immediate -1 需要立刻执行。
- UserBlocking 250ms 超时时间250ms，一般指的是用户交互。
- Normal 5000ms 超时时间5s，不需要直观立即变化的任务，比如网络请求。
- Low 10000ms 超时时间10s，肯定要执行的任务，但是可以放在最后处理。
- Idle 一些没有必要的任务，可能不会执行

react通过类似 requestIdleCallback 去向浏览器做一帧一帧请求，等到浏览器有空余时间，去执行异步更新任务，来保证页面的流畅度

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cdece5756244975beb3ca5352af4eb8~tplv-k3u1fbpfcp-watermark.awebp)

#### 模拟requestIdleCallback

requestIdleCallback只有谷歌浏览器支持，为了兼容性，react实现了自己的requestIdleCallback。模拟实现requestIdleCallback需要具备两个条件

- 可以主动让出主线程，让浏览器去渲染视图

- 一次事件循环只执行一次，因为执行一个以后，还会请求下一次的时间

有两个满足条件的宏任务

1. setTimeout

不使用setTimeout是因为递归执行的时候，其回调函数的触发会有一定程度的延迟

```js
let time = 0 
let nowTime = +new Date()
let timer
const poll = function(){
    timer = setTimeout(()=>{
        const lastTime = nowTime
        nowTime = +new Date()
        console.log( '递归setTimeout(fn,0)产生时间差：' , nowTime -lastTime )
        poll()
    },0)
    time++
    if(time === 20) clearTimeout(timer)
}
poll()
```

2. MessageChannel

为了让视图流畅地运行，可以按照人类能感知到最低限度每秒 60 帧的频率划分时间片，这样每个时间片就是 16ms 。也就是这 16 毫秒要完成如上 js 执行，浏览器绘制等操作，而上述 setTimeout 带来的浪费就足足有 4ms，看一下MessageChannel是如何触发宏任务的

```js
let scheduledHostCallback = null 
  /* 建立一个消息通道 */
  var channel = new MessageChannel();
  /* 建立一个port发送消息 */
  var port = channel.port2;

  channel.port1.onmessage = function(){
      /* 执行任务 */
      scheduledHostCallback() 
      /* 执行完毕，清空任务 */
      scheduledHostCallback = null
  };
  /* 向浏览器请求执行更新任务 */
  requestHostCallback = function (callback) {
    scheduledHostCallback = callback;
    if (!isMessageLoopRunning) {
      isMessageLoopRunning = true;
      port.postMessage(null);
    }
  };
```

在一次更新任务中，react调用requestHostCallback，port2项port1发起消息通知，port1的onmessage触发，执行异步更新任务

#### 调度流程

1. [react concurrent](https://zhuanlan.zhihu.com/p/60307571?utm_source=tuicool)