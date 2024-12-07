
---
title: "BVH | HLBVH建树方法"
date: 2024-12-07T07:36:04Z
draft: false
categories: [ "CG", "pbrt"]
isCJKLanguage: true
slug: "92aa5f22"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
# UEVersion: 5.3.2 
---

#  HLBVH

HLBVH为什么快: 普通BVH建树是自顶向下，必须要知道上一层的bounds大小才能继续建树。而HLBVH通过莫顿编码可以快速的划分不同Primitives到不同的cluster,这样允许并行独立的处理不同cluster,然后再合并成BVH.

优点：
- 允许一定程度的并行建树，速度快

缺点：
- 利用 Morton 码能够二分空间的性质来建树，每一层二分的算法实质上等于按最简单的二分轴的方式，质量很差

## morton编码的一些性质

PBRT是这样描述的:
> which map nearby points in dimensions to nearby points along the 1D line, where there is an obvious ordering function. After the primitives have been sorted, spatially nearby clusters of primitives are in contiguous segments of the sorted array. 

莫顿编码能够把高维数据映射到一维，并且在空间上相邻的点在莫顿编码上也是相邻的。

二维编码的莫顿编码是一个类似Z字的形状。

![edit-47cbf9d610a84cf29e6dcc996caf42f9-2024-12-01-13-10-14](https://img.blurredcode.com/img/edit-47cbf9d610a84cf29e6dcc996caf42f9-2024-12-01-13-10-14.png?x-oss-process=style/compress)

莫顿编码有另外一个很重要的性质，是最高位的bit可以将空间划分为两个部分。

比如上图中，上半部分的第一个bit始终等于1，下半部分的第一个bit始终等于0。根据不同bit的分布关系我们很容易构造出不断二分的空间格子划分。


## Morton Code Encode / Decode 编码 Primitive坐标

morton编码位数有限，所以相比于直接encode世界坐标，我们可以encode 每个Primitive相对于Bounds的偏移量([0,1]的值域)。

```cpp
std::vector<MortonPrimitive> mortonPrims(primitiveInfo.size());
ParallelFor(
    [&](int i) {
        <<Initialize mortonPrims[i] for ith primitive>> 
            constexpr int mortonBits = 10;
            constexpr int mortonScale = 1 << mortonBits;
            mortonPrims[i].primitiveIndex = primitiveInfo[i].primitiveNumber;
            Vector3f centroidOffset = bounds.Offset(primitiveInfo[i].centroid); // CentroidOffset介于[0,1]之间，是一个相对于Bounds的值
            mortonPrims[i].mortonCode = EncodeMorton3(centroidOffset * mortonScale);

    }, primitiveInfo.size(), 512);
```



32位 3维 Encode

可以按照`z10y10x10...z3y3x3 z2y2x2 z1y1x1`这个顺序来排布，每个维度有10个bit来编码。

从虚幻里抠了一部分代码里出来
```cpp
	/** Spreads bits to every 3rd. */
	static constexpr FORCEINLINE uint32 MortonCode3( uint32 x )
	{
		x &= 0x000003ff;
		x = (x ^ (x << 16)) & 0xff0000ff;
		x = (x ^ (x <<  8)) & 0x0300f00f;
		x = (x ^ (x <<  4)) & 0x030c30c3;
		x = (x ^ (x <<  2)) & 0x09249249;
		return x;
	}

	/** Reverses MortonCode3. Compacts every 3rd bit to the right. */
	static constexpr FORCEINLINE uint32 ReverseMortonCode3( uint32 x )
	{
		x &= 0x09249249;
		x = (x ^ (x >>  2)) & 0x030c30c3;
		x = (x ^ (x >>  4)) & 0x0300f00f;
		x = (x ^ (x >>  8)) & 0xff0000ff;
		x = (x ^ (x >> 16)) & 0x000003ff;
		return x;
	
```

如果想要更高的精度，可以用64位来encode.
每个维度有20个bit来编码，总共60bit。
比较适合大世界，会有更高的精度。
算一下，如果一个8kmx8km的地图，假设BVH的Bounds覆盖整个地图，那么用32位来编码，一个轴只有10个bit 1024,也就是8米的范围。而64位编码可以精确到`0.0078`米，这个精度已经足够了。

```cpp
// BVHAccel Utility Functions (64-bit version)
inline uint64_t MortonCode3_64(uint64_t x) {
    //CHECK_LE(x, (1ULL << 20));  // 1 << 20 for 64-bit (to stay within 64-bit range)
    if (x == (1ULL << 20)) --x;
    if (x >= (1ULL << 21)) x = (1ULL << 21) - 1;

    x = (x | (x << 32)) & 0x1F00000000FFFF;
    x = (x | (x << 16)) & 0x1F0000FF0000FF;
    x = (x | (x << 8))  & 0x100F00F00F00F00F;
    x = (x | (x << 4))  & 0x10C30C30C30C30C3;
    x = (x | (x << 2))  & 0x1249249249249249;
    return x;
}

inline uint64_t MortonCode3_64_Reverse(uint64_t x) {
    x &= 0x1249249249249249;
    x = (x ^ (x >> 2)) & 0x10C30C30C30C30C3;
    x = (x ^ (x >> 4)) & 0x100F00F00F00F00F;
    x = (x ^ (x >> 8)) & 0x1F0000FF0000FF;
    x = (x ^ (x >> 16)) & 0x1F00000000FFFF;
    x = (x ^ (x >> 32)) & 0x1FFFFF;
    return x;
}
```


## 排序Mordon码

std::sort倒也不是不行..mordon码这种纯数字的排序对基数排序更友好一点。

pbrt直接给了一个 radix sort 的实现，可以参考一下。
一次 6 个bit，`2^6 = 64`个 bucket，排序 30 位的 morton 码需要 5 个 Pass。
如果要排序 60位的 morton 码可以考虑考虑多加点桶，少遍历几次。

## 划分Primitives clusters

取高位12bit，只要高位12bit有一个值不一样，则划分为不同的cluster
总共会划分出4096个cluster
每个cluster都是三维的，每个轴各占4bit，每个轴16个cell(相当于我们把整个空间切成了4096 (16 * 16 * 16)份)

实践中这里会稀疏的。

```cpp
    // Find intervals of primitives for each treelet
    std::vector<LBVHTreelet> treeletsToBuild;
    for (int start = 0, end = 1; end <= (int)mortonPrims.size(); ++end) {
        // > a set of points with Morton codes that match in their high bit values lie in a power-of-two aligned and sized subset of the original volume.
#ifdef PBRT_HAVE_BINARY_CONSTANTS
      uint32_t mask = 0b00111111111111000000000000000000;
#else
      uint32_t mask = 0x3ffc0000;
#endif
      if (end == (int)mortonPrims.size() ||
            ((mortonPrims[start].mortonCode & mask) !=
             (mortonPrims[end].mortonCode & mask))) {
            // Add entry to _treeletsToBuild_ for this treelet
            int nPrimitives = end - start;
            int maxBVHNodes = 2 * nPrimitives;
            BVHBuildNode *nodes = arena.Alloc<BVHBuildNode>(maxBVHNodes, false);
            treeletsToBuild.push_back({start, nPrimitives, nodes});
            start = end;
        }
```


### 并行向下建子树

将空间划分为 4096 个 cluster 后，我们能够确定哪些 Primitive 落在这些不同的 cluster 中。
这个时候可以继续自顶向下建树，并行线程。

```cpp
    // Create LBVHs for treelets in parallel
    std::atomic<int> atomicTotal(0), orderedPrimsOffset(0);
    orderedPrims.resize(primitives.size());
    ParallelFor([&](int i) {
        // Generate _i_th LBVH treelet
        int nodesCreated = 0;
        const int firstBitIndex = 29 - 12;
        LBVHTreelet &tr = treeletsToBuild[i];
        tr.buildNodes =
            emitLBVH(tr.buildNodes, primitiveInfo, &mortonPrims[tr.startIndex],
                     tr.nPrimitives, &nodesCreated, orderedPrims,
                     &orderedPrimsOffset, firstBitIndex);
        atomicTotal += nodesCreated;
    }, treeletsToBuild.size());
    *totalNodes = atomicTotal;
```


这里一定要记得:
- Morton码从高位往低位遍历时候，每一个位都可以将空间从中间划分为两部分，（等价于 二分切分的 BVH 建造方法）
- 每一个 Primitive 的 Morton 码可以根据最高位的 bit 来确定是在空间的上半部分还是下半部分


PBRT这里有一个小的优化，因为 primitives 都是有序的，如果所有的 primitives 某一层空间的同一侧，那么这一层不用划分，直接继续往下走下一个 bit，直到找到一个 bit 使得这一层的 primitives 被划分到两侧，可以减少树的深度。


```cpp
        // Figure 4.8: Implications of the Morton Encoding.
        // 认真读这个图
        // morton码的最高位bit 相当于对半分开了整个空间
        // 设置了1的在上面，0在下面
        // 如果全在同一侧，那么这一层不用划分
        int mask = 1 << bitIndex;
        // Advance to next subtree level if there's no LBVH split for this bit
        if ((mortonPrims[0].mortonCode & mask) ==
            (mortonPrims[nPrimitives - 1].mortonCode & mask))
            return emitLBVH(buildNodes, primitiveInfo, mortonPrims, nPrimitives,
                            totalNodes, orderedPrims, orderedPrimsOffset,
                            bitIndex - 1);

        // Find LBVH split point for this dimension
        // 通过 morton 码二分搜搜查找划分点
        int searchStart = 0, searchEnd = nPrimitives - 1;
        while (searchStart + 1 != searchEnd) {
            CHECK_NE(searchStart, searchEnd);
            int mid = (searchStart + searchEnd) / 2;
            if ((mortonPrims[searchStart].mortonCode & mask) ==
                (mortonPrims[mid].mortonCode & mask))
                searchStart = mid;
            else {
                CHECK_EQ(mortonPrims[mid].mortonCode & mask,
                         mortonPrims[searchEnd].mortonCode & mask);
                searchEnd = mid;
            }
        }
        int splitOffset = searchEnd;
        // Create and return interior LBVH node
        (*totalNodes)++;
        BVHBuildNode *node = buildNodes++;
        BVHBuildNode *lbvh[2] = {
            emitLBVH(buildNodes, primitiveInfo, mortonPrims, splitOffset,
                     totalNodes, orderedPrims, orderedPrimsOffset,
                     bitIndex - 1),
            emitLBVH(buildNodes, primitiveInfo, &mortonPrims[splitOffset],
                     nPrimitives - splitOffset, totalNodes, orderedPrims,
                     orderedPrimsOffset, bitIndex - 1)};
```


### 以 cluster 为叶子节点建立 SAH BVH

注意到我们这里cluster 都是独立的，我们可以继续将其整合为一个完整的 BVH 树
最后将所有的cluster合并成一个BVH树。


这里实际上就是一个普通的 SAH 建树过程。
只是建树的对象从原来的 Primitive 变成了 cluster。

- 先把所有的的 cluster 作为叶子节点，然后总体的 Bounds
- 自顶向下建树，每次选择一个最优的划分点(计算 cost)，然后递归的建立左右子树
- 直到所有的 cluster 都落在一个合适的叶子节点

```cpp
    // Create and return SAH BVH from LBVH treelets
    std::vector<BVHBuildNode *> finishedTreelets;
    finishedTreelets.reserve(treeletsToBuild.size());
    for (LBVHTreelet &treelet : treeletsToBuild)
        finishedTreelets.push_back(treelet.buildNodes);
    return buildUpperSAH(arena, finishedTreelets, 0, finishedTreelets.size(),
                         totalNodes);
```
