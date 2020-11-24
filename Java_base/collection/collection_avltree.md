## Java集合基础: 平衡二叉树(AVL Tree)

我们之前学习了二叉排序树, 它很好的平衡了插入与查找的效率, 但是不平衡的二叉排序树效率很低。AVL就是解决二叉树不平衡的问题。

### 1. AVL定义

`平衡二叉树(Self-Balancing Binary Search Tree或Height-Banlanced Binary Search Tree)`是一种二叉排序树, 其中每一个节点的左子树和右子树高度最多为1。

它是一种高度平衡的二叉排序树。也就是说, 要么它是一颗空树, 要么它的左子树和右子树都是平衡二叉树, 并且左子树和右子树的深度之差的绝对值不超过1。

二叉树上节点的左子树深度减去右子树深度的值称为**平衡因子BF(Balance Factor),**那么平衡二叉树上所有节点的平衡因子只可能是-1,0,1。

![noavl](/image/noavl.png)

上图就不是一个AVL树, 因为节点18的左子树高度为2, 右子树的高度为0, 高度差大于1。

我们可以通过一定的手段调整为, 平衡二叉树:

![avl1](/image/avl1.png)

### 2. 实现原理

我们从上图可以知道, 当我们在插入新节点之后, 需要遵守二叉排序树的规则, 但是又不能出现不平衡的情况。**所以就需要通过平衡因子来判断是否新节点会导致父节点或者祖节点"不平衡"。**

平衡二叉树构建的基本思想就是在二叉排序树插入节点时, 先检查是否破坏了树的平衡性, 如果是, 则找出**最小不平衡子树**。在保证二叉排序树的特性前提下(也就是左子树都小于根节点, 右子树都大于根节点), 调整最小不平衡子树中各节点之间的连接关系, 进行相应的旋转, 使之成为新的平衡子树。

> **最小不平衡子树**是指距离插入节点最近的, 并且平衡因子的绝对值大于1的节点为根的子树。

在代码中, 就是在递归的插入元素时, 递归判断每一个节点的平衡因子, 递归的维护平衡, 从而使得整棵树始终保持平衡。这种方式就是**左旋和右旋**。

#### 2.1 右旋

**当插入的节点在不平衡节点的左侧和左侧(LL)时, 我们利用右旋在实现平衡。**

如下图, 插入一个新节点之后出现不平衡:

![noavl2](/image/noavl2.png)

根据二叉排序树的性质, 它们之间的大小关系为:

> T1 < z < T2 < x < T3 < y < T4

此时我们发现, 它的最小不平衡子树就是以y为根节点的树。

右旋就是将x节点的右子树先拿掉, 然后让x节点的右孩子等于y节点, 最后y节点的左孩子等于刚刚拿掉的x节点的右孩子。

![avl_ll_1](/image/avl_ll_1.png)
![avl_ll_2](/image/avl_ll_2.png)
![avl_ll_3](/image/avl_ll_3.png)

可以看到, 经过右旋之后, 二叉树重新恢复平衡, 并且上面的大小关系也没有发生改变。

而且添加节点的过程是递归, 所以右旋之后的子树的父节点和祖节点也会进行相应的右旋或者左旋知道达到平衡状态。

右旋代码如下:

```java
/**
 * LL
 *
 * / 对节点进行右旋转操作，返回右旋转之后新的根节点
 * /        y                            x
 * /       / \                         /   \
 * /      x   T4     向右旋转(y)      z     y
 * /     / \       -------------->  / \   /  \
 * /    z  T3                      T1 T2 T3  T4
 * /   / \
 * /  T1 T2
 */
private Node rightRotate(Node y) {
    Node x = y.left;
    // 保存x节点的右子树，即使右子树为空，也没关系
    Node t3 = x.right;

    // 右旋
    x.right = y;
    // 将原本x的右子树放在y的左子树的位置
    y.left = t3;

    // 更新height
    y.height = Math.max(getHeight(y.left), getHeight(y.right)) + 1;
    x.height = Math.max(getHeight(x.left), getHeight(x.right)) + 1;
    return x;
}
```

#### 2.2 左旋

**当插入元素在第一个不平衡节点的右侧的右侧(RR), 我们就可以利用左旋来实现平衡。**

其实左旋跟右旋是对称的, 直接上代码:

```java
/**
 * RR
 * <p>
 * / 对节点进行左旋转操作，返回左旋转之后新的根节点
 * /        y                            x
 * /       / \                         /   \
 * /      T1  x     向左旋转(y)       y     z
 * /         / \    -------------->  / \   / \
 * /        T2  z                   T1 T2 T3 T4
 * /           / \
 * /          T3 T4
 */
private Node leftRotate(Node y) {
    Node x = y.right;
    Node t2 = x.left;

    // 右旋
    x.left = y;
    y.right = t2;

    //更新height
    y.height = Math.max(getHeight(y.left), getHeight(y.right)) + 1;
    x.height = Math.max(getHeight(x.left), getHeight(x.right)) + 1;
    return x;
}
```

#### 2.3 LR

**当插入的元素在不平衡节点的左侧的右侧(LR), 我们可以先进行左旋, 使其转化为插入的元素在不平衡节点的左侧的左侧(LL), 在进行右旋即可。**

![avl_lr_1](/image/avl_lr_1.png)

对x节点,进行左旋之后就转化为LL的情况, 按照第一种情况进行右旋处理即可。

![avl_lr_2](/image/avl_lr_2.png)

#### 2.4 RL

**当插入的元素在不平衡节点的右侧的左侧(RL), 我们可以先进行右旋, 使其转化为插入的元素在不平衡节点的右侧的右侧(LL), 在进行左旋即可。**

这种情况正好与LR对称。

#### 2.5 总结

上面的四种情况都是利用了左旋和右旋的方式来实现平衡。具体代码如下:

```java
public void put(K key, V value) {
    Node newNode = new Node(key, value);
    root = put(root, newNode);
}

/*** 递归算法：插入一个元素，返回插入新节点后树的根 */
private Node put(Node node, Node newNode) {
    if (node == null) {
        size++;
        return newNode;
    }

    if (newNode.key.compareTo(node.key) < 0) {
        node.left = put(node.left, newNode);
    } else if (newNode.key.compareTo(node.key) > 0) {
        node.right = put(node.right, newNode);
    } else {
        node.value = newNode.value;
    }

    // 更新高度
    node.height = Math.max(getHeight(node.left), getHeight(node.right)) + 1;

    // 维护平衡
    int balanceFactor = getBalanceFactor(node);

    // LL
    if (balanceFactor > 1 && getBalanceFactor(node.left) >= 0) {
        // 右旋转
        return rightRotate(node);
    }

    // LR
    if (balanceFactor > 1 && getBalanceFactor(node.left) < 0) {
        // 先对当前节点的左孩子进行左旋转转变为LL，然后在进行右旋转
        node.left = leftRotate(node.left);
        // 右旋转
        return rightRotate(node);
    }

    // RR
    if (balanceFactor < -1 && getBalanceFactor(node.right) <= 0) {
        // 左旋转
        return leftRotate(node);
    }

    //RL
    if (balanceFactor < -1 && getBalanceFactor(node.right) > 0) {
        // 先对当前节点的右孩子进行右旋转转变为RR，然后在进行左旋转
        node.right = rightRotate(node.right);
        // 左旋转
        return leftRotate(node);
    }

    return node;
}
```

[平衡二叉树参考文章](https://www.jianshu.com/p/4f5eca987990)





