---
title: 缓存淘汰策略-LRU
date: 2021-04-15 13:06:49
tags:
---



## 分析

LRU总体大概是这样的，最近使用的放在前面，最近没用的放在后面，如果来了一个新的数，此时内存满了，就需要把旧的数淘汰。

- 为了方便移动数据，使用链表类似的数据结构
- 要判断这条数据是不是最新的或者最旧的，使用hashmap等key-value形式的数据结构。
- 如果数据量特别大，

## 第一种实现(LinkedHashMap)

```
public class LRUCache {

    int capacity;
    Map<Integer,Integer> map;

    public LRUCache(int capacity){
        this.capacity = capacity;
        map = new LinkedHashMap<>();
    }

    public int get(int key){
        //如果没有找到
        if (!map.containsKey(key)){
            return -1;
        }
        //找到了就刷新数据
        Integer value = map.remove(key);
        map.put(key,value);
        return value;
    }

    public void put(int key,int value){
        if (map.containsKey(key)){
            map.remove(key);
            map.put(key,value);
            return;
        }
        map.put(key,value);
        //超出capacity，删除最久没用的即第一个,或者可以复写removeEldestEntry方法
        if (map.size() > capacity){
            map.remove(map.entrySet().iterator().next().getKey());
        }
    }

    public static void main(String[] args) {
        LRUCache lruCache = new LRUCache(10);
        for (int i = 0; i < 10; i++) {
            lruCache.map.put(i,i);
            System.out.println(lruCache.map.size());
        }
        System.out.println(lruCache.map);
        lruCache.put(10,200);
        System.out.println(lruCache.map);
    }
```

![Image](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhq3OnECT6N8PXoicxEXBE8EEk6WeYaPqabgIkcj0adzxGz0gABSwzJEIyMbZz2g2WicL5HgeXNNMlzA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 第二种实现(双链表+hashmap)

```
public class LRUCache {

    private int capacity;
    private Map<Integer,ListNode>map;
    private ListNode head;
    private ListNode tail;

    public LRUCache2(int capacity){
        this.capacity = capacity;
        map = new HashMap<>();
        head = new ListNode(-1,-1);
        tail = new ListNode(-1,-1);
        head.next = tail;
        tail.pre = head;
    }

    public int get(int key){
        if (!map.containsKey(key)){
            return -1;
        }
        ListNode node = map.get(key);
        node.pre.next = node.next;
        node.next.pre = node.pre;
        return node.val;
    }

    public void put(int key,int value){
        if (get(key)!=-1){
            map.get(key).val = value;
            return;
        }
        ListNode node = new ListNode(key,value);
        map.put(key,node);
        moveToTail(node);

        if (map.size() > capacity){
            map.remove(head.next.key);
            head.next = head.next.next;
            head.next.pre = head;
        }
    }

    //把节点移动到尾巴
    private void moveToTail(ListNode node) {
        node.pre = tail.pre;
        tail.pre = node;
        node.pre.next = node;
        node.next = tail;
    }

}
```

像第一种方式，如果复写removeEldestEntry会更简单，这里简单的展示一下

```
public class LRUCache extends LinkedHashMap<Integer,Integer> {
    private int capacity;
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity;
    }
}
```

