diff算法用于对新旧虚拟DOM树对比，差异更新，尽可能少地操作DOM。传统的diff算法通过循环递归节点依此比较，复杂度O(n^3)，n是树中节点的个数，显然开销太大了，react通过以下的策略将复杂度变为O(n)

- dom节点跨层级操作较少，忽略不计（tree diff优化策略）

- 拥有相同类的两个组件将会生成相似的树形结构，拥有不同类的两个组件将会生成不同的树形结构（component diff优化策略）

- 对于同一层级的一组子节点，它们可以通过唯一 id 进行区分（element diff优化策略）

diff的过程分别以下几个步骤

#### tree diff

由于dom节点跨层级操作较少可以忽略不计，对新旧DOM树同一层级地节点进行比较，只会对同一父节点下的子节点进行比较，如果该节点不存在，则不会进一步地对子节点进行比较

![](https://pic2.zhimg.com/80/v2-815bb35800d89ef234a78a6a54f80c0d_1440w.jpg)

如果出现了跨层级的dom操作，react只考虑同层级节点的位置变换，对于不同层级的节点只有创建和删除操作

如下，当发现第二层级少了节点A，直接删除节点A，第三层多了节点A，重新创建节点A，所以尽量不要进行跨层级的dom操作

![](https://pic1.zhimg.com/80/v2-51c77dd83a0aabfc23c67e31154e7c14_1440w.jpg)

#### component diff

diff过程中遇到组件时，如果是同一类型的组件，继续进行该组件的tree diff或element diff，否则，由于不同的组件很少会存在相似的dom结构，于是会替换组件下所有子节点

如下，当发现组件D变更为了组件G时，会删掉组件D及其子节点，然后创建组件G及其子节点

![](https://pic1.zhimg.com/80/v2-257306f11432a6ee3ef333ee3f7c3f28_1440w.jpg)

#### element diff

当节点处于同一层级时，依赖唯一的key值进行节点位置变换，包括插入、移动、删除节点操作，示例如下，其中变化前后相同的节点key不变。指针lastIndex初始为0，与相同节点在变换之前的索引位置mountIndex相比，如果lastIndex较大，则对该节点进行移动，否则更新lastIndex为mountIndex，依次类推

示例一： A B C D -> B A D C

1. lastIndex = 0，nextIndex = 0，B之前的索引是1且大于lastIndex，令lastIndex = 1，nextIndex ++
2. lastIndex = 1，A之前的索引是0小于lastIndex，移动A到nextIndex位置，nextIndex ++
3. lastIndex = 1，D之前的索引是3大于lastIndex，令lastIndex = 3，nextIndex ++
4. lastIndex = 3，C之前的索引是2小于lastIndex，移动C到nextIndex位置

![](https://pic4.zhimg.com/80/v2-227f0c761b0505d26615a29cc4a2a65b_1440w.jpg)

示例二：A B C D -> B E C A

1. lastIndex = 0，nextIndex = 0，B之前的索引是1且大于lastIndex，令lastIndex = 1，nextIndex ++
2. lastIndex = 1，E是新节点，创建E，lastIndex不变，nextIndex ++
3. lastIndex = 1，C之前的索引是2且大于lastIndex，令lastIndex = 2，nextIndex ++
4. lastIndex = 2，A之前的索引是0且小于lastIndex，移动A到nextIndex位置
5. 对老集合进行循环遍历，判断是否存在新集合中没有但老集合中仍存在的节点，发现是D节点，删除D

![](https://pic3.zhimg.com/80/v2-8088513e0eafdf5527f71317478493b6_1440w.jpg)

示例三：A B C D -> D A B C

1. lastIndex = 0，nextIndex = 0，D之前的索引为3且大于lastIndex，令lastIndex = 3，nextIndex ++
2. lastIndex = 3，A之前的索引为0且小于lastIndex，移动A到nextIndex位置，nextIndex ++
3. lastIndex = 3，B之前的索引为1且小于lastIndex，移动B到nextIndex位置，nextIndex ++
4. lastIndex = 3，C之前的索引为2且小于lastIndex，移动C到nextIndex位置，nextIndex ++

![](https://pic3.zhimg.com/80/v2-4aa0583a59de45f6b67cd5847b279a6a_1440w.jpg)

这是diff算法不足的地方，理论上 diff 应该只需对 D 执行移动操作，然而由于 D 在老集合的位置是最大的，导致其他节点的 _mountIndex < lastIndex，造成A、B、C 全部移动到 D 节点后面

[简易代码实现](https://github.com/livoras/simple-virtual-dom)

参考
1. [React探索-diff算法](https://zhuanlan.zhihu.com/p/103187276)