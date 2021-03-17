diff算法用于对新旧虚拟DOM树对比，差异更新，尽可能少地操作DOM

#### tree diff

对新旧DOM树同一层级法地节点进行比较，只会对同一父节点下的子节点进行比较，如果该节点不存在，则不会进一步地对器子节点进行比较

#### component diff

如果是同一类型的组件，进行虚拟dom树的tree diff或element diff，否则替换组件下所有子节点

> 对于同一类型的组件，如果react事先知道了虚拟dom树没有发生任何变化，可以节省大量diff运算的时间，可以使用shouldComponentUpdate来决定组件是否需要diff

#### element diff

不加key值时，例如一个组件内的四个元素a,b,c,d，更改为b,a,d,c，对应位置进行对比，发现都不相同，会全部进行删除并重新创建

加key值后，还是上面的例子，发现更改后的key值都存在，只进行真实DOM元素的插入、移动、删除就可以了

lastIndex相当于一个标记，默认为0，当元素索引大于lastIndex时，不移动并lastIndex改变为这个索引值；否则，移动元素至lastIndex位置

例如由a,b,c,d更改为b,a,e,d,c（添加了key值）

1. 初始lastIndex为0，从左向右比较更改后的节点在更改前内容中的索引
2. b之前的索引为1，lastIndex < 1，b不动，令lastIndex=1
3. a之前的索引为0，lastIndex > 0，a移动到索引为1（lastIndex）的位置，现顺序为b,a,c,d
4. e为新增节点，令lastIndex=2，e插入到索引为2（lastIndex）的位置，现顺序为b,a,e,c,d
5. d之前的索引为3，lastIndex < 3，d不动，令lastIndex=3+1(加上新增的节点)
6. c之前的索引为2，lastIndex > 2，c移动到索引为4（lastIndex）的位置，现顺序为b,a,e,d,c

#### 代码实现

(简易代码实现)[https://github.com/livoras/simple-virtual-dom]