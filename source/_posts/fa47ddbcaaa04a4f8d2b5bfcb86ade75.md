---
layout: post
title: 如何理解Kotlin函数式编程特性？
abbrlink: fa47ddbcaaa04a4f8d2b5bfcb86ade75
tags:
  - kotlin
categories:
  - Mac笔记本
  - kotlin
date: 1715847897367
updated: 1726804750349
---

- 命令式：改变状态、分支、赋值\
  ![480758e818fcfa359835ec5eedf07de4.png](/resources/5a5bdca1515a4e4fb9aa5d68d9aeeac4.png)
- 函数式：\
  ![5069eb89c5de2205d81c56de432508fc.png](/resources/80b78708ec5246b0a0ff40dbd4dc269b.png)
  - 避免可变状态
  - 程序是表达式和变换关系，不是命令
  - 建立在数学直觉上
- 权责让渡
  - 将低层次的细节控制权交给运行时，开发者只关注高层次细节
- 函数副作用
  - 执行时，除了返回值，还对外部产生了其他影响，eg:修改了全局变量、外部变量
  - 纯函数：没有副作用，对应数学中的函数

## 函数式编程模式

- `三板斧`代替迭代

  - `filter`、`map`、`reduce`,`fold` （压缩）

- 闭包\
  ![649da5b9432f3d1c962039ed85f30895.png](/resources/de69b21d17484435a2b81e0de146fb46.png)

  - 带缓存：\
    ![9b7d16382c06952f5ee5396fbd46793b.png](/resources/0dbd30bbcf7e499fa6a5d475544dd45f.png)
  - `记忆`
  - 柯里化：函数部分施用，提高函数灵活性，分布求解，权责让渡。
  - eg:构造url\
    ![276415aa37afa9a6ab6143883ba11a2f.png](/resources/863a8678a2b14a15b6a378e7b5135d81.png)

- 函数式编程模式\
  ![0ac70d3efb63b20f4304caca0d2b161f.png](/resources/b98ebb93cd3b4832850daf9d15462db2.png)

- 尾递归优化\
  ![e366e3f705836b4696212b6f9e9c8d94.png](/resources/9141039691614221b0de3f1ec4948e30.png)

- 异常

- 不相交结合体，且无需关注else属性

![7bf7ca0a195f9299c24554d51dc02669.png](/resources/dd84890adfd943c1af32436bb7b77294.png)

![5bc3efa77b6e6c5609c182e3544cfc97.png](/resources/c760813843fb4efc906d14d12da8bc9f.png)

![443309f781c802517fc16b1dfe8c46b4.png](/resources/833cc44d567f4450b716683ddfbc9b5a.png)

![ff3e574fce3017678c36742de4268e6e.png](/resources/525a2226f62c420ca6e08433633f071c.png)

## Kotlin函数式编程

1. Lambda表达式：与Java类似也支持Lambda表达式。
2. 高阶函数：函数作为参数，或返回值
3. 内联函数：`inline`减少运行时开销

### 编译后的字节码

Kotlin编译器处理Lambda表达式时，会生成一个匿名类，但实现方式比Java更优化。具体老说，Kotlin会尽量避免生成不必要的类和对象

#### 内联函数

显著减少Lambda表达式开销。编译时会将函数直接替换到调用处，避免匿名类的生成

## 对比Java

1. 性能优化：通过内联减少Lambda表达式运行时开销。Java依赖于`invokedynamic`指令和`LambdaMetafactory`优化Lambda表达式执行

2. 语法简洁

3. 生成字节码：Kotlin也会生成匿名类，但实际应用中，Kotlin会尽量优化，减少不必要的类生成，

   1. inline
   2. lambda表达式优化：SAM转换：

   ```Kotlin
   //SAM转换：对于单一抽象方法，Kotlin会将	Lambda表达式转为该接口的实例，无需显式创建匿名类
       val runnable = Runnable { 					println("Running") }

   ```

   ```
    3.使用`invokedynamic`
    4.字节码优化，避免不必要的对象创建。数组操作转换为循环
   ```

kotlin :Object: 静态类实例的成员方法,成员变量\
seal CLASS:抽象类
