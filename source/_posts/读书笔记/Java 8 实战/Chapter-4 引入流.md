---
title: 'chapter4-引入流'
date: 2021-10-04 18:24:28
tags: [system design]
---

## 流是什么

- 以声明的方式（通过查询语句表达，而不是临时写实现）处理数据集合

- 遍历数据的高级迭代器

## 集合与流

### 集合和流比较

- 无法并行处理了元素



java使用集合处理数据

```
List<Dish> lowCaloricDishes = new ArrayList();
for(Dish : dishes){
	if(dishes.getCalories() < 400)
		lowCaloricDishes.add(dish)
}

Collections.sort(lowCaloricDishes, (d1, d2) -> return d1.getCalories() - d2.getCaloriest());

List<String> lowCaloricDishNames = new ArrayList();
for(Dish d : lowCaloricDishes){
  lowCaloricDishNames.add(d.getName());
}
```

java使用流处理同样的数据

```
List<String> lowCaloricDishNames =  dishes
.parallelStream()
.filter(d->d.getCalories() < 400)
.sorted(comparing(Dishes::getCalories))
.map(e->e.getName())
.collect(toList())
```

总结stream api的优势：

- 声明性：更简单、更透明
- 可复合：更灵活
- 可并行：性能更好



### 流简介

简短定义：从支持数据处理操作的源生成的元素序列

- 元素序列：就像集合一样，流也提供了一个接口，可以访问特定元素类型的一组有序值。因为集合是数据结构，所以它的主要目的是以特定的时间/空间复杂度存储和访问元素（如ArrayList 与 LinkedList）。但**流的目的在于表达计算**，比如filter、 sorted和map。集合讲的是数据，流讲的是计算。
- 源：流会使用一个提供数据的源，如集合、数组或输入/输出资源。 请注意，从有序集合生成流时会保留原有的顺序。由列表生成的流，其元素顺序与列表一致。
- 数据处理操作：流的数据处理功能支持类似于数据库的操作，以及函数式编程语言中的常用操作，如filter、 map、 reduce、 find、 match、 sort等。流操作可以顺序执行，也可并行执行。
- 流水线：很多流操作本身会返回一个流，这样多个操作就可以链接起来，形成一个大的流水线，流水线的操作可以看作对数据源进行数据库式查询。
- 内部迭代：与使用迭代器显式迭代的集合不同，流的迭代操作是在背后进行的。



### 流与集合

Java现有的集合概念和新的流概念都提供了接口，来配合代表元素型有序值的数据接口。所谓有序，就是说我们一般是按顺序取用值，而不是随机取用的。

粗略地说，集合与流之间的差异就在于什么时候进行计算。

- 集合是一个内存中的数据结构，它包含数据结构中目前所有的值，集合中的每个元素都得先算出来才能添加到集合中。（你可以往集合里加东西或者删东西，但是不管什么时候，集合中的每个元素都是放在内存里的，元素都得先算出来才能成为集合的一部分。）
- 相比之下，流则是在概念上固定的数据结构（你不能添加或删除元素），其元素则是按需计算的。 这对编程有很大的好处。这个思想就是用户仅仅从流中提取需要的值，而这些值在用户看不见的地方，只会按需生成。这是一种生产者，消费者的关系。从另一个角度来说，流就像是一个延迟创建的集合，只有在消费者要求的时候才会计算值（用管理学的话说这就是需求驱动，甚至是实时制造)。



#### 只能遍历一次

请注意，和迭代器类似，流只能遍历一次。遍历完之后，我们就说这个流已经被消费掉了。你可以从原始数据源那里再获得一个新的流来重新遍历一遍，就像迭代器一样（这里假设它是集合之类的可重复的源，如果是I/O通道就没戏了）。

#### 外部迭代与内部迭代

使用Collection接口需要用户去做迭代（比如用for-each），这称为外部迭代。 相反，Streams库使用内部迭代，它帮你把迭代做了，还把得到的流值存在了某个地方，你只要给出一个函数说要干什么就可以了。



### 流操作

#### 常用中间操作

操作	类型	返回类型	操作参数	函数描述符
filter	中间	Stream<T>	Predicate<T>	T -> boolean
map	中间	Stream<R>	Function<T, R>	T -> R
limit	中间	Stream<T>		
sorted	中间	Stream<T>	Comparator<T>	(T, T) -> int
distinct	中间	Stream<T>		



#### 常用终端操作

操作	类型	目的
forEach	终端	消费流中的每个元素并对其应用 Lambda。这一操作返回 void
count	终端	返回流中元素的个数。这一操作返回 long
collect	终端	把流归约成一个集合，比如 List、 Map 甚至是 Integer。








