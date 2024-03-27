.vue文件是无法直接运行在浏览器中的，以webpack为例，会通过vue-loader将.vue文件编译为js代码。我们主要关注运行时vue做了什么事情

#### 挂载阶段

![](../assets/vue-mount.webp)

#### 更新阶段

##### 依赖收集

##### 派发更新