---
title: 'union-find算法'
date: 2021-05-28 10:59:50
tags:
---

## union-find算法的API

union-find算法用于检测动态连通性，例如计算机网络中的两个节点是否连通，在一个特定圈子里的两个人是否有间接的朋友关系，等等。

<img src="https://upload-images.jianshu.io/upload_images/15884954-de7ef738949aad50.png" alt="img" style="zoom:67%;" />

对于图的动态连接性问题，我们最关心的问题是：图中节点间之间是否存在路径（节点是否处于同一个连通分量中）。对于图的动态变化，我们允许动态地添加边，但是不允许删除边。因此，定义API如下：

| API签名                        | 描述                        |
| :----------------------------- | :-------------------------- |
| `bool connected(int p, int q)` | 节点`p`与节点`q`是否连通    |
| `void union(int p, int q)`     | 将节点`p`与节点`q`连接起来  |
| `int find(int p)`              | 寻找`p`处于哪一个连通分量中 |





```java
public class UF {
 private int id[]; 
 private int count; //连通分量数量

 public UF(int N) {
     count = N;
     id = new int[N];
     for (int i = 0; i < N; i++) 
         id[i] = i;
 }

 public int count() {
	return count;
 }

 public boolean connected(int p, int q) {
 	return find(p) == find(q);
 }

 public int find(int p); 
    
 public void union(int p, int q);
    
}
```



## union-find算法的实现1：quick-find

id[] 表示的事连通分量id,以节点为索引

quick-find算法是union-find算法的众多实现中最简单也最没有效率的一种，它的主要实现如下：

```java
public int find(int p) {
  return id[p];
}

public void union(int p, int q) {
  // 查找点 p 和点 q 在id数组中的值，从而可以接着判断它们是否在相同的连通分量中
  int pID = find(p);
  int qID = find(q);

  // p, q已经在相同的分量中，无需任何操作
  if (pID == qID) {
    return;
  }

  // 执行到这里，说明 p 和 q 不在相同的分量中，因此需要把它们进行归并，在这里，是把 p 所在的分量里的所有元素全部重命名为 q 所在的分量里的名称（唯一）
  for (int i = 0; i < id.length; i++) {
    if (id[i] == pID)  id[i] = qID;
  
  count--;
}
```

find() 函数用于查找一个点 p 的名称；union() 函数用于归并两个点 p 和 q，如果它们已经在同一个连通分量里，那么不会产生任何归并效果。



> 之所以把这种算法叫quick-find，是因为它的 find() 操作很快，只需要访问id数组一次；
>
> 但是quick-find的 union() 操作却很慢，因为它需要访问整个id数组。



> 命题F:  归并2个连接分量的操作，访问数组的次数在( N+3，2N+1)，其中N是点的个数。
>
> union() 函数一开始的两个 find() 操作无论如何是逃不掉的，所以这里至少就访问了 2 次id数组。因此，最后的for循环应该是至少访问了 N+1 次数组，但这又是怎么算出来的呢？id[i] == qID 至少会被执行一次，所以for循环访问数组的次数就至少是 N+1 次了。
>
> 当id数组中，除了q之外，其他所有元素都与p处于同一连通分量中的话，那么，if (id[i] == pID) 这个条件就会成立 N-1 次，这意味着 id[i] = qID 会被执行 N-1 次，而N次 if 判断无论如何都是会被执行的，所以for循环里访问数组的次数是 (N-1)+ (N-1) + 1=2N-1 次，另外前面说了，union()函数中两个find()操作是免不了的，所以还要再加2次，总共是 2N-1+2=2N+1 次。





## union-find算法的实现2：quick-union

<!--考虑一下，为什么以上的解法会造成“牵一发而动全身”？因为每个节点所属的组号都是单独记录，各自为政的，没有将它们以更好的方式组织起来，当涉及到修改的时候，除了逐一通知、修改，别无他法。什么样子的数据结构能够将一些节点给组织起来？常见的就是链表，图，树，什么的了。但是哪种结构对于查找和修改的效率最高？毫无疑问是树，因此考虑如何将节点和组的关系以树的形式表现出来。-->

为了提高union()的速度，它和quick-find算法互补。它也是基于相同的数据结构 - 以触点作为索引的id[]数组。但是意义已经不同了，采用parent-link的方式将节点组织起来，举例而言，id[p]的值就是p节点的父节点的序号，如果p是树根的话，id[p]的值就是p，因此最后经过若干次查找，一个节点总是能够找到它的根节点，即满足id[root] = root的节点也就是组的根节点了，然后就可以使用根节点的序号来表示组号。所以在处理一个pair的时候，将首先找到pair中每一个节点的组号(即它们所在树的根节点的序号)，如果属于不同的组的话，就将其中一个根节点的父节点设置为另外一个根节点，相当于将一颗独立的树编程另一颗独立的树的子树

![img](https://pic4.zhimg.com/80/v2-44540ed217035c0d23167324ddf4b063_720w.jpg)

```
private int find(int p){
	while(p != id[p]) p = id[p]; //找出连接分量的名称，即根节点名称
	return p;
}

public void union(int p, int q){
	int pRoot = find(p);
	int qRoot = find(q);
	if(pRoot == qRoot) return;
	
	id[pRoot] = iqRoot;  //将p和q的根节点连接在一起
	
	count--;
}
```





## union-find算法的实现3：加权quick-union

uick-union算法有一个明显的缺点，就是会出现一个极端情况：

![img](https://pic4.zhimg.com/80/v2-3e0f32ff867519251c1ec7d81b1f010b_720w.jpg)

在这种情况中，树的深度会非常大，最大可以达到N-1，显然这样遍历树就很不划算，于是解决办法是：判断数的深度，并总是将小树连接到大树上。

![img](https://pic2.zhimg.com/80/v2-1e31170eb5b80af6d2f91704c080410d_720w.jpg)



```java
public class WeightedQuickUnionUf{

private int[] id;
private int[] sz; //各个根节点所对应的分量的大小，以触点为索引
private int count;

public WeightedQuickUnionUf(int N){
	count = N;
	id = new int[N];
	for(int i=0; i<N; i++) id[i] = i;
	sz = new int[N];
	for(int i=0; i<N; i++) sz[i] = 1;
}
private int find(int p){
	while(p != id[p]) p = id[p]; //找出连接分量的名称，即根节点名称
	return p;
}



public void union(int p, int q){
	int i = find(p);
	int j = find(q);
	if(i == j) return;
	
    //将小树挂在大树上
	if( sz[i] < sz[j]){
		id[i] = j;
		sz[j] += sz[i];
	}else{
        id[j] = i;
		sz[i] += sz[j];
    }
	
	count--;
}


```

> 命题：森林中大小为K的树的高度<=lgK
>
> 证明：归纳法：当k=1，高度<=lg1；假设当k=i, 高度 <= lgi; 那么当i<=j, 并且 i + j =k‘,  我们将小树i和大树归并，那么小树中所有节点深度+1, 大树节点深度不变，而归并后的树k'大小为i+j = k’, 而1+lgi= lg(i+i) <= lg(i+j) = lgk‘; 得证。

![img](https://pic4.zhimg.com/80/v2-aa15e51d568de64766f60ffd28f0b443_720w.jpg)

![img](https://pic2.zhimg.com/80/v2-39d79a38fd9116c76cb552dfdb9bd9a9_720w.jpg)
