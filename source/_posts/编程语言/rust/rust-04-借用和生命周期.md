---
title: wasm
date: 2022-12-08 18:39:15
tags:
---



## 1. 所有权和借用

我们已经知道，使用变量"a"赋值给变量"b"，有2种情况

- 他们的类型是可复制的，即它实现了clone trait 和 copy trait。2个变量拥有2个不同的对象。这种情况就是**"复制语义"。**
- 他们的类型是不可复制，即没有实现copy trait。只有一个变量（从a到b）拥有这个对象。这种情况就是**"移动语义"。**



上述情况都可以保证内存的安全性，但是，如果使用了**引用**，情况就变得复杂

```rust
let n = 12;
let ref_to_n = &n
```

n和ref_to_n引用了同一个对象，**但只有n有【所有权】，ref_to_n可以访问，但是无【所有权】，这是【借用】**

关于可变性，

```rust
let mut n = 12;
let ref1_to_n = &mut n; //可变借用，只能从可变变量执行可变借用
let ref2_to_n = &n;  //不可变借用
```



## 2. 对象生命周期

【作用域】只适用于编译时变量，而不适用于运行时的对象；运行时的对象的相应概念，是【生命周期】

### 2者的开始时间不同

```rust
let a; //变量a的作用域开始
a = 12; //对象12的生命周期开始
print!(a);
```

### 2者的结束时间不同

```rust
let mut a = "hello".to_string() //a的作用域开始，"hello"的对象生命周期开始
let mut b = a;  //a的作用域暂停，b的作用域开始，因为a所有权移动到b了，a不再能被访问
print!(b) //错误 print!(a); 
a = "cat".to_string(); //a的作用域恢复，"cat"的对象生命周期开始
print!(a)
b = a // a的作用域暂停, b从前拥有的"hello"对象被丢弃，生命周期结束，如果实现了'drop'trait，则被调用;

//首先b的作用域退出, 然后是a
//b拥有的'cat'对象被丢弃，生命周期结束
```





## 3.关于借用的错误

### 错误1：丢弃后使用

```rust
let ref_to_n;
{
	let n = 12; //n变量作用域开始，12这个对象生命周期开始，并且在栈中
  ref_to_n = &n; //ref_to_n借用12
  print!(*ref_to_n)
}
//失败 print!(*ref_to_n) //n doesn't live enough
```

line7: n的变量死亡，其拥有的对象也已经丢弃，但是存在 ref_to_n还引用这个对象。因为ref_to_n的生命周期应该比对象12少一点



c++就无法避免这种错误

```c++
int* ref_to_n;
{
	let n = 12; 
  ref_to_n = &n; 
  print!(*ref_to_n)
}
print!(*ref_to_n)
```



#### 避免错误
