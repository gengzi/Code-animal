[TOC]

## 目标

* 了解 java8 stream 的操作

  参考：java8 实战书籍，java简明教程

## Stream

对于stream 不要误认为是操作io 的类，java8 提供的stream 是用于处理集合(Collection)的数据流。

### 基础概念

通过stream 可以实现链式操作，以 点 方法 的形式，将方法链接的在一起。

```java
Arrays.stream(new String[]{"aaaa","b","cc"}).sorted().limit(2).collect(Collectors.toList());
```

将数据一步步流转到每个方法执行，最终得到需要的结果。使用stream 可以看做使用sql 语句来处理数据，方法的调用顺序不同，可能会影响整理数据的效率。在sql语句的查询中，应当在最内层限定查询条件，缩小结果集。

在链式方法调用中，分为两种方法，一种是  中转方法，一种是 终止方法。

中转方法返回的结果对象都是 Stream 数据流对象，终止方法 用来终止Stream 数据流的操作。

上述的 sorted ，limit 都是中转方法，collect 是终止方法

### 注意点

参考：java简明教程

多数数据流操作都接受一些lambda表达式参数，函数式接口用来指定操作的具体行为。这些操作的大多数必须是无干扰而且是无状态的。它们是什么意思呢？

当一个函数不修改数据流的底层数据源，它就是无干扰的。例如，在上面的例子 中，没有任何lambda表达式通过添加或删除集合元素修改 myList 。

当一个函数的操作的执行是确定性的，它就是无状态的。例如，在上面的例子中， 没有任何lambda表达式依赖于外部作用域中任何在操作过程中可变的变量或状态。

我的理解：

在stream 数据流，处理数据时，不要修改底层的数据，比如 添加，删除 操作，最好也不要修改底层数据，或者使用一些外部可变参数来控制执行过程，因为你有时候会忽略这些操作，导致一些莫名其妙的结果出现。

Stream 提供了并行操作，通过 parallelStream() 来开启，利用多核cpu来提供执行速度。

其中针对Stream 中提供的大多数方法，参数都是 函数式接口（只有一个抽象方法的接口），

比如 Function、 Predicate 、Consumer、ToIntFunction 等等。

## 具体的api 参考

### Stream 的相关api

![tIyuGQ.png](https://s1.ax1x.com/2020/06/10/tIyuGQ.png)

### collectors 相关

![tIynPg.png](https://s1.ax1x.com/2020/06/10/tIynPg.png)

![tIyK2j.png](https://s1.ax1x.com/2020/06/10/tIyK2j.png)

 

## 小技巧

针对使用 Stream 提供的方法，使用的参数都是函数式接口，有时候我们并不清楚，要怎么实现，依然可以使用 匿名函数的形式，new xxx(){}  来创建，实现其中的逻辑，通过idea 的提示，将其转换为 lamdba 表达式的形式。

在使用 Stream 的方法时，注意不同方法的实现顺序，应该选择最优的方式，减少方法的调用提高执行效率。

 

 

 