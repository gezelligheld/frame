在 render 阶段更新 Fiber 节点时，通过对比当前 dom 对应的 current Fiber 树和当前最新的 jsx 对象去构建 workInProgress Fiber 树，

diff 算法用于对新旧虚拟 DOM 树对比，差异更新，尽可能少地操作 DOM。传统的 diff 算法通过循环递归节点依此比较，复杂度 O(n^3)，n 是树中节点的个数，显然开销太大了，react 通过以下的策略将复杂度变为 O(n)

- 只对同级比较，跨层级的 dom 不会进行复用（tree diff 优化策略）

- 不同类型节点生成的 dom 树不同，此时会直接销毁老节点及子孙节点，并新建节点（component diff 优化策略）

- 对于同一层级的一组子节点，它们可以通过唯一 id 进行区分（element diff 优化策略）

#### 单节点 diff

单节点 diff 有以下几种情况

- key 和 type 相同表示可以复用节点

- key 不同直接标记删除节点，然后新建节点

- key 相同 type 不同，标记删除该节点和兄弟节点，然后新创建节点

```js
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement
): Fiber {
  const key = element.key;
  let child = currentFirstChild;

  //child节点不为null执行对比
  while (child !== null) {
    // 1.比较key
    if (child.key === key) {
      // 2.比较type

      switch (child.tag) {
        //...

        default: {
          if (child.elementType === element.type) {
            // type相同则可以复用 返回复用的节点
            return existing;
          }
          // type不同跳出
          break;
        }
      }
      //key相同，type不同则把fiber及和兄弟fiber标记删除
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      //key不同直接标记删除该节点
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  //新建新Fiber
}
```

#### 多节点 diff

##### tree diff

由于 dom 节点跨层级操作较少可以忽略不计，对新旧 DOM 树同一层级地节点进行比较，只会对同一父节点下的子节点进行比较，如果该节点不存在，则不会进一步地对子节点进行比较

![](https://pic2.zhimg.com/80/v2-815bb35800d89ef234a78a6a54f80c0d_1440w.jpg)

如果出现了跨层级的 dom 操作，react 只考虑同层级节点的位置变换，对于不同层级的节点只有创建和删除操作

如下，当发现第二层级少了节点 A，直接删除节点 A，第三层多了节点 A，重新创建节点 A，所以尽量不要进行跨层级的 dom 操作

![](https://pic1.zhimg.com/80/v2-51c77dd83a0aabfc23c67e31154e7c14_1440w.jpg)

##### component diff

diff 过程中遇到组件时，如果是同一类型的组件，继续进行该组件的 tree diff 或 element diff，否则，由于不同的组件很少会存在相似的 dom 结构，于是会替换组件下所有子节点

如下，当发现组件 D 变更为了组件 G 时，会删掉组件 D 及其子节点，然后创建组件 G 及其子节点

![](https://pic1.zhimg.com/80/v2-257306f11432a6ee3ef333ee3f7c3f28_1440w.jpg)

##### element diff

当节点处于同一层级时，依赖唯一的 key 值进行节点位置变换，包括插入、移动、删除节点操作，示例如下，其中变化前后相同的节点 key 不变。指针 lastIndex 初始为 0，与相同节点在变换之前的索引位置 mountIndex 相比，如果 lastIndex 较大，则对该节点进行移动，否则更新 lastIndex 为 mountIndex，依次类推

示例一： A B C D -> B A D C

1. lastIndex = 0，nextIndex = 0，B 之前的索引是 1 且大于 lastIndex，令 lastIndex = 1，nextIndex ++
2. lastIndex = 1，A 之前的索引是 0 小于 lastIndex，移动 A 到 nextIndex 位置，nextIndex ++
3. lastIndex = 1，D 之前的索引是 3 大于 lastIndex，令 lastIndex = 3，nextIndex ++
4. lastIndex = 3，C 之前的索引是 2 小于 lastIndex，移动 C 到 nextIndex 位置

![](https://pic4.zhimg.com/80/v2-227f0c761b0505d26615a29cc4a2a65b_1440w.jpg)

示例二：A B C D -> B E C A

1. lastIndex = 0，nextIndex = 0，B 之前的索引是 1 且大于 lastIndex，令 lastIndex = 1，nextIndex ++
2. lastIndex = 1，E 是新节点，创建 E，lastIndex 不变，nextIndex ++
3. lastIndex = 1，C 之前的索引是 2 且大于 lastIndex，令 lastIndex = 2，nextIndex ++
4. lastIndex = 2，A 之前的索引是 0 且小于 lastIndex，移动 A 到 nextIndex 位置
5. 对老集合进行循环遍历，判断是否存在新集合中没有但老集合中仍存在的节点，发现是 D 节点，删除 D

![](https://pic3.zhimg.com/80/v2-8088513e0eafdf5527f71317478493b6_1440w.jpg)

示例三：A B C D -> D A B C

1. lastIndex = 0，nextIndex = 0，D 之前的索引为 3 且大于 lastIndex，令 lastIndex = 3，nextIndex ++
2. lastIndex = 3，A 之前的索引为 0 且小于 lastIndex，移动 A 到 nextIndex 位置，nextIndex ++
3. lastIndex = 3，B 之前的索引为 1 且小于 lastIndex，移动 B 到 nextIndex 位置，nextIndex ++
4. lastIndex = 3，C 之前的索引为 2 且小于 lastIndex，移动 C 到 nextIndex 位置，nextIndex ++

![](https://pic3.zhimg.com/80/v2-4aa0583a59de45f6b67cd5847b279a6a_1440w.jpg)

这是 diff 算法不足的地方，理论上 diff 应该只需对 D 执行移动操作，然而由于 D 在老集合的位置是最大的，导致其他节点的 \_mountIndex < lastIndex，造成 A、B、C 全部移动到 D 节点后面

多节点 diff 相关源码如下，可以看到有三次 for 循环

```js
//ReactChildFiber.old.js

function reconcileChildrenArray(
  returnFiber: Fiber, //父fiber节点
  currentFirstChild: Fiber | null, //childs中第一个节点
  newChildren: Array<*>, //新节点数组 也就是jsx数组
  lanes: Lanes //lane相关 第12章介绍
): Fiber | null {
  let resultingFirstChild: Fiber | null = null; //diff之后返回的第一个节点
  let previousNewFiber: Fiber | null = null; //新节点中上次对比过的节点

  let oldFiber = currentFirstChild; //正在对比的oldFiber
  let lastPlacedIndex = 0; //上次可复用的节点位置 或者oldFiber的位置
  let newIdx = 0; //新节点中对比到了的位置
  let nextOldFiber = null; //正在对比的oldFiber
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    //第一次遍历
    if (oldFiber.index > newIdx) {
      //nextOldFiber赋值
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }
    const newFiber = updateSlot(
      //更新节点，如果key不同则newFiber=null
      returnFiber,
      oldFiber,
      newChildren[newIdx],
      lanes
    );
    if (newFiber === null) {
      if (oldFiber === null) {
        oldFiber = nextOldFiber;
      }
      break; //跳出第一次遍历
    }
    if (shouldTrackSideEffects) {
      //检查shouldTrackSideEffects
      if (oldFiber && newFiber.alternate === null) {
        deleteChild(returnFiber, oldFiber);
      }
    }
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx); //标记节点插入
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  if (newIdx === newChildren.length) {
    deleteRemainingChildren(returnFiber, oldFiber); //将oldFiber中没遍历完的节点标记为DELETION
    return resultingFirstChild;
  }

  if (oldFiber === null) {
    for (; newIdx < newChildren.length; newIdx++) {
      //第2次遍历
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      if (newFiber === null) {
        continue;
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx); //插入新增节点
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
    return resultingFirstChild;
  }

  // 将剩下的oldFiber加入map中
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

  for (; newIdx < newChildren.length; newIdx++) {
    //第三次循环 处理节点移动
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx],
      lanes
    );
    if (newFiber !== null) {
      if (shouldTrackSideEffects) {
        if (newFiber.alternate !== null) {
          existingChildren.delete(
            //删除找到的节点
            newFiber.key === null ? newIdx : newFiber.key
          );
        }
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx); //标记为插入的逻辑
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }

  if (shouldTrackSideEffects) {
    //删除existingChildren中剩下的节点
    existingChildren.forEach((child) => deleteChild(returnFiber, child));
  }

  return resultingFirstChild;
}
```

第一次遍历处理了以下情况

1. key 不同，第一次循环结束
2. newChildren 或者 oldFiber 遍历完，第一次循环结束
3. key 同 type 不同，标记 oldFiber 为 DELETION（删除 tag）
4. key 相同 type 相同则可以复用

第二次循环处理了以下情况

1. newChildren 和 oldFiber 都遍历完，多节点 diff 结束
2. newChildren 没遍历完，oldFiber 遍历完，将剩下的 newChildren 的节点标记为 Placement（插入 tag）
3. newChildren 和 oldFiber 没遍历完，则进入节点移动的逻辑

第三次循环主要处理节点移动，逻辑主要在 placeChild 中

```js
function placeChild(newFiber, lastPlacedIndex, newIndex) {
  newFiber.index = newIndex;

  if (!shouldTrackSideEffects) {
    return lastPlacedIndex;
  }

  var current = newFiber.alternate;

  if (current !== null) {
    var oldIndex = current.index;

    if (oldIndex < lastPlacedIndex) {
      //oldIndex小于lastPlacedIndex的位置 则将节点插入到最后
      newFiber.flags = Placement;
      return lastPlacedIndex;
    } else {
      return oldIndex; //不需要移动 lastPlacedIndex = oldIndex;
    }
  } else {
    //新增插入
    newFiber.flags = Placement;
    return lastPlacedIndex;
  }
}
```

参考

1. [React 探索-diff 算法](https://zhuanlan.zhihu.com/p/103187276)
2. [简易代码实现](https://github.com/livoras/simple-virtual-dom)
3. [react 源码解析 9.diff 算法](https://xiaochen1024.com/courseware/60b1b2f6cf10a4003b634718/60b1b354cf10a4003b634721)
