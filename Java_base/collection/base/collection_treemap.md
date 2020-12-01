## Java集合分析: TreeMap

`TreeMap`是**红黑树**的Java实现, 所以了解**红黑树**的原理很重要。后面的`HashMap`也是如此。

**红黑树**能保证增, 删, 查等基本操作的时间复杂度**O(lgN)**

我们先看下`TreeMap`的结构:

![collection_treemap](/image/collection_treemap.png)