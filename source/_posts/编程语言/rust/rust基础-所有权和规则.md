---
title: wasm
date: 2022-12-08 18:39:15
tags:
---





### Value in a scope

#### Rule 1: 

- You can have only one owner in a scope; When the owner of the value left the scope, the value droped;
- When passing a value to another scope, the owner moves to this scope;

```rust
{
  let arr = vec![1]; // this scope owns vec![1]
  let arr2= vec![2];
  sum(arr);  //move occors so sum is the new owner of value vec![1]
  println!(arr); // fail, 
} //arr2 dropped the value vec1[2]
```

#### Rule 2:

- A value can have multipe immutable referrences

```rust
{
  let arr = vec![1]; // this scope owns vec![1] 
  sum(&arr); //1st borrower: sum scope
  max(&arr); //2nd borrower: max scope
}
```

