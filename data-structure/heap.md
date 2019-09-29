# heap

## 完全二叉树
>
> 除最后一层外，每一层上的结点数均达到最大值；在最后一层上只缺少右边的若干结点
>
> 出于简便起见,完全二叉树通常采用数组而不是链表存储

## 原理

> 完全二叉树是效率很高的数据结构，**堆是一种完全二叉树或者近似完全二叉树**，所以效率极高，像十分常用的排序算法、Dijkstra算法、Prim算法等都要用堆才能优化，几乎每次都要考到的二叉排序树的效率也要借助平衡性来提高，而平衡性基于完全二叉树

1. 大根堆, 根结点的键值是所有堆结点键值中最大者
2. 小根堆, 根结点的键值是所有堆结点键值中最小者

> 插入第i个元素的所用的时间是 `O(log i)`，所以插入所有元素的整体时间复杂度是`O(NlogN)`

## 过程

1. 先构建完成二叉树, 调整成大/小根堆
2. 将根节点和最后一个子节点交换

## 实现

参考 PriorityQueue.class

``` java
// 初始化 小根堆
// 1. 堆的特性保证 size>>>1 下方只有一层
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--) {
        siftDown(i, queue[i]);
    }
}

// 堆特性保证 从 half 开始转换小根堆, 只需要找出子节点 最小节点
// 将最小节点 和 half 做比较, 如果 小于 half, 则做一次交换即可
private void siftDown(int i, Integer x) {
    int half = size >>> 1;
    while (i < half) {
        int left = (i << 1) + 1;
        Integer o = queue[left];
        int right = left + 1;
        // 选出最小的 节点值, 兄弟节点, left 存放最小节点下标
        if (right < size && o > queue[right]) {
            o = queue[right];
            left = right;
        }
        // 父节点与最小值对比
        if (x < o) {
            break;
        }
        queue[i] = o;
        i = left;
    }
    queue[i] = x;
}

// 堆特性保证 从 half 开始转换小根堆, 只判断父节点是否比加入节点大
// 目标节点只要在最后出一次即可
private void siftUp(int i, Integer x) {
    while (i > 0) {
        int parent = (i - 1) >>> 1;
        Integer o = queue[parent];
        if (x >= o) {
            break;
        }
        queue[i] = o;
        i = parent;
    }
    queue[i] = x;
}
```
