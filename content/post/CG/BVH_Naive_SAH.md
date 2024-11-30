
---
title: "BVH | Naive和SAH建树方法"
date: 2024-11-30T18:41:41+08:00
draft: false
categories: [ "CG"]
isCJKLanguage: true
slug: "0e73fe60"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
katex: true
markup: mmark
mmarktoc: true
# UEVersion: 5.3.2 
---

# Naive 建树方法


## 二分Bounds法

每次取得BVH节点的最长轴，将其Box沿着最长轴一分为二，落在不同子节点的Primitive分别放入两个子节点中。

```cpp
    case SplitMethod::Middle: {
        // Partition primitives through node's midpoint
        Float pmid = (centroidBounds.pMin[dim] + centroidBounds.pMax[dim]) / 2;
        auto midIter = std::partition(bvhPrimitives.begin(), bvhPrimitives.end(),
                                        [dim, pmid](const BVHPrimitive &pi) {
                                            return pi.Centroid()[dim] < pmid;
                                        });
        mid = midIter - bvhPrimitives.begin();
        // For lots of prims with large overlapping bounding boxes, this
        // may fail to partition; in that case do not break and fall through
        // to EqualCounts.
        if (midIter != bvhPrimitives.begin() && midIter != bvhPrimitives.end())
            break;
    }
```

pbrt这个`std::partition`用的恰到好处，刚好可以用一个条件来分割一个range为两个cluster。
实质上它是快排实现的其中一步，快排每次选择一个pivot以后都需要把pivot两侧的数据放置到pivot两侧，这个过程就是partition。


## 二分数量法

```cpp
case SplitMethod::EqualCounts: {
    // Partition primitives into equally sized subsets
    mid = bvhPrimitives.size() / 2;
    std::nth_element(bvhPrimitives.begin(), bvhPrimitives.begin() + mid,
                        bvhPrimitives.end(),
                        [dim](const BVHPrimitive &a, const BVHPrimitive &b) {
                            return a.Centroid()[dim] < b.Centroid()[dim];
                        });

    break;
```            

应该没什么人会用的方法，就是让两边子节点在数量上比较平衡。
pbrt这里也是标准库的函数用的比较惊艳，`std::nth_element`可以部分排序一个集体，Leetcode经常出现的`TopK`问题就可以用这个解决。


# SAH BVH

PBRT的默认建树方式，核心目标是让两边子树的**求交代价**保持尽可能接近。求交代价实际上和Primitive的表面积和数量有关。

最好的参考资料:
http://15462.courses.cs.cmu.edu/fall2017/lecture/acceleratingqueries/slide_025


![edit-47cbf9d610a84cf29e6dcc996caf42f9-2024-07-20-23-50-01](https://img.blurredcode.com/img/edit-47cbf9d610a84cf29e6dcc996caf42f9-2024-07-20-23-50-01.png?x-oss-process=style/compress)

对于一次求交的代价来说，

$$C = C_trav+ p(A)N_A C_{Isect} +  p(B)N_B C_{Isect}$$

$$N_{A} / N_{B}$$子树A/B中的物体数量

$$C_{Isect}$$求交的计算代价，可以认为是一个常数

`C_Trav`和树的深度有关系


可以看到，一次求交来说，最重要是要

$$min(p_{A}N_A + p_{B}N_B)$$


![edit-47cbf9d610a84cf29e6dcc996caf42f9-2024-07-20-23-53-47](https://img.blurredcode.com/img/edit-47cbf9d610a84cf29e6dcc996caf42f9-2024-07-20-23-53-47.png?x-oss-process=style/compress)

根据最后这一页PPT，
可以得到

最小的代价为

$$\frac{S_AN_A + S_BN_B}{S_N}$$

-  $$S_N$$是确定的

-  存在某个平面用于分割两个子树，使得$$S_AN_A + S_BN_B$$最小


算法：

- 沿最长轴设立N个Bucket(假设8个)
- 以三角形中心点，计算这个三角形属于哪个Bucket
- 新建一个数组cost[N]，遍历过去，依次计算假设以这个Bucket为分割点时的代价$$S_AN_A + S_BN_B$$


## 建树方法

Step1: 计算所有的Primitive落在哪个Bucket， PBRT用了12个Bucket
```cpp
    PBRT_CONSTEXPR int nBuckets = 12;
    BucketInfo buckets[nBuckets];

    // Initialize _BucketInfo_ for SAH partition buckets
    for (int i = start; i < end; ++i) {
        int b = nBuckets *
            // Offset方法: 计算一个点在Bounds的相对位置，p = pMax , offset = 1, p = pMin, offset = 0
                centroidBounds.Offset(
                    primitiveInfo[i].centroid)[dim];
        if (b == nBuckets) b = nBuckets - 1;
        buckets[b].count++;
        buckets[b].bounds =
            Union(buckets[b].bounds, primitiveInfo[i].bounds);
    }
```

Step2： 新建代价数组，计算以每个Bucket分割时的代价

```cpp
// Compute costs for splitting after each bucket
// 注意这是个O(N^2)的搜索
Float cost[nBuckets - 1];
for (int i = 0; i < nBuckets - 1; ++i) {
    Bounds3f b0, b1;
    int count0 = 0, count1 = 0;
    for (int j = 0; j <= i; ++j) {
        b0 = Union(b0, buckets[j].bounds);
        count0 += buckets[j].count;
    }
    for (int j = i + 1; j < nBuckets; ++j) {
        b1 = Union(b1, buckets[j].bounds);
        count1 += buckets[j].count;
    }
    cost[i] = 1 +
                (count0 * b0.SurfaceArea() +
                count1 * b1.SurfaceArea()) /
                    bounds.SurfaceArea();
}
```


Step3: 分割Primitive

```cpp
if (nPrimitives > maxPrimsInNode || minCost < leafCost) {
    BVHPrimitiveInfo *pmid = std::partition( //部分排序数组，使得满足条件的在前面
        &primitiveInfo[start], &primitiveInfo[end - 1] + 1,
        [=](const BVHPrimitiveInfo &pi) {
            int b = nBuckets *
                    centroidBounds.Offset(pi.centroid)[dim];
            if (b == nBuckets) b = nBuckets - 1;
            return b <= minCostSplitBucket;
        });
    mid = pmid - &primitiveInfo[0];
```


# Cache友好的BVH： 树状结构转数组

BVH遍历的时候经常需要从父节点遍历到很深的子节点，一个简单的优化就是把树状结构转到Flatten的数组中，这样可以减少Cache Miss。

这个题也是Leetcode的常见题了。


LeetCode的常客了.. 二叉树转数组表示(前序遍历)
[144. 二叉树的前序遍历 - 力扣（LeetCode）](https://leetcode.cn/problems/binary-tree-preorder-traversal/)
```
     A 
   /  \
  B    E     --> [A B C D E]
 /\               4 3           // 右节点的偏移量
/ \
C  D
```

每个节点只需要记录它的右子树的偏移量，(如果有左子树，必定是下一个元素)
在BVH里不会出现某个节点只有右叶子的情况..

```cpp

root = BuildBVH(...);
nodes = new LinearBVHNode[totalNodes];
int offset = 0;
flattenBVH(root, &offset);

int BVHAggregate::flattenBVH(BVHBuildNode *node, int *offset) {
    LinearBVHNode *linearNode = &nodes[*offset];
    linearNode->bounds = node->bounds;
    int nodeOffset = (*offset)++;
    if (node->nPrimitives > 0) {
        CHECK(!node->children[0] && !node->children[1]);
        CHECK_LT(node->nPrimitives, 65536);
        linearNode->primitivesOffset = node->firstPrimOffset;
        linearNode->nPrimitives = node->nPrimitives;
    } else {
        // Create interior flattened BVH node
        linearNode->axis = node->splitAxis;
        linearNode->nPrimitives = 0;
        flattenBVH(node->children[0], offset);
        linearNode->secondChildOffset = flattenBVH(node->children[1], offset);
    }
    return nodeOffset;
}
```


# 并行建树

PbrtV4相较于Pbrtv3的代码里看起来增加了一点点的并行建树的支持，但不多。
真正要比较好的支持并行建树的要HLBVH比较好。

现在对于`Primitives`比较多的情况下，两个子树会分散在两个线程里建树（同理递归的时候也有一定程度的并行)
```cpp
BVHBuildNode *children[2];
// Recursively build BVHs for _children_
if (bvhPrimitives.size() > 128 * 1024) {
    // Recursively build child BVHs in parallel
    ParallelFor(0, 2, [&](int i) {
        if (i == 0)
            children[0] = buildRecursive(
                threadAllocators, bvhPrimitives.subspan(0, mid), totalNodes,
                orderedPrimsOffset, orderedPrims);
        else
            children[1] =
                buildRecursive(threadAllocators, bvhPrimitives.subspan(mid),
                                totalNodes, orderedPrimsOffset, orderedPrims);
    });

} else {
    // Recursively build child BVHs sequentially
    children[0] =
        buildRecursive(threadAllocators, bvhPrimitives.subspan(0, mid),
                        totalNodes, orderedPrimsOffset, orderedPrims);
    children[1] =
        buildRecursive(threadAllocators, bvhPrimitives.subspan(mid),
            
```