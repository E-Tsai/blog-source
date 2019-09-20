---
title: lambda-calculus
date: 2018-02-28 14:24:58
tags: programming-language
categories: cs
---



关于[λ演算](https://zh.wikipedia.org/zh-sg/%CE%9B%E6%BC%94%E7%AE%97)的一些学习笔记。

[我的最爱Lambda演算——开篇 · cgnail's weblog](https://link.zhihu.com/?target=http%3A//cgnail.github.io/academic/lambda-1/)

[阿隆佐.丘奇的天才之作——lambda演算中的数字 · cgnail's weblog](https://link.zhihu.com/?target=http%3A//cgnail.github.io/academic/lambda-2/)

[Lambda演算中的布尔值和选择 · cgnail's weblog](https://link.zhihu.com/?target=http%3A//cgnail.github.io/academic/lambda-3/)

[为什么是Y？ · cgnail's weblog](https://link.zhihu.com/?target=http%3A//cgnail.github.io/academic/lambda-4/)

[从Lambda演算到组合子演算 · cgnail's weblog](https://link.zhihu.com/?target=http%3A//cgnail.github.io/academic/lambda-5/)

[Lambda演算的类型 · cgnail's weblog](https://link.zhihu.com/?target=http%3A//cgnail.github.io/academic/lambda-6/)

[终章，Lambda演算建模——程序即证明！ · cgnail's weblog](https://link.zhihu.com/?target=http%3A//cgnail.github.io/academic/lambda-7/)

 <!-- more -->

## 演算规则

### Alpha转换

如果有这样一个表达式：

```
lambda x . if (= x 0) then 1 else x ^ 2 
```

我们可以用Alpha转换，将`x`变成`y`（写作`alpha[x / y]`），于是我们有：

```
lambda y . if (= y 0) then 1 else y ^ 2 
```

### Beta规则

“ `(lambda x y. x y) (lambda z . z * z) 3` “。这是一个有两个参数的函数，它(的功能是)把第一个参数应用到第二个参数上。当我们运算时，我们替换第一个函数体中的参数“`x`”为“`lambda z . z * z` “；然后我们用“`3`”替换参数“`y`”，得到：“ `(lambda z . z * z) 3` “。 再执行Beta规约，有“`3 * 3`”。

Beta规则的形式化写法为：

```
lambda x . B e = B[x := e] if free(e) subset free(B[x := e]) 
```

最后的条件“`if free(e) subset free(B[x := e])`”说明了为什么我们需要Alpha转换：我们只有在不引起绑定标识符和自由标识符之间的任何冲突的情况下，才可以做Beta规约：如果标识符“`z`”在“`e`”中是自由的，那么我们就需要确保，Beta规约不会导致“`z`”变成绑定的。如果在“`B`”中绑定的变量和“`e`”中的自由变量产生命名冲突，我们就需要用Alpha转换来更改标识符名称，使之不同。



## 数字

