[TOC]

## 目标

* 了解lamdba 表达式

  参考 java 简明教程文档

  [深入理解Java双冒号(::)运算符的使用](https://blog.csdn.net/zhoufanyang_china/article/details/87798829)

  [Java8新特性2：方法引用--深入理解双冒号::的使用](https://blog.csdn.net/aigoV/article/details/102586953)

## 基础概念

引入两个概念: 函数式接口，接口中的默认方法

* 函数式接口：如果一个接口中，只包含一个抽象方法，就可以认为是一个函数式接口。其中可以使用注解

  @FunctionalInterface 标识这个接口是一个函数式接口，当然不使用这个注解标识也是可以的。

  ```java
  /**
   * 函数式接口，只包含一个抽象方法
   */
  @FunctionalInterface
  public interface MyPrint<T> {
      String output(T str);
  }
  ```

* 默认方法

  默认方法用于扩展接口中的方法，jdk8 之后，为了在接口中引入其他的功能，需要在接口中提供额外的方法，不能直接在原来的接口中，添加方法，这样会导致实现该接口的类，都要做修改，所以就引入了默认方法，使用 default 标识。默认方法是一个非抽象的方法，实现该接口的类，也都继承默认方法。也可以在接口中添加静态方法，用来扩展接口的功能。

  ```java
  // List 接口中的 sort 就是一个默认方法
  @SuppressWarnings({"unchecked", "rawtypes"})
      default void  sort(Comparator<? super E> c) {
          Object[] a = this.toArray();
          Arrays.sort(a, (Comparator) c);
          ListIterator<E> i = this.listIterator();
          for (Object e : a) {
              i.next();
              i.set((E) e);
          }
      }
 
    // Comparator 接口的中的 静态方法
      public static <T extends Comparable<? super T>> Comparator<T> reverseOrder() {
          return Collections.reverseOrder();
      }
  ```

 

## lamdba 表达式

语法：(param1，param2) -> { return xxx  }

首先，函数式接口，才可以用lamdba 表达式，来表示。函数式接口中的默认方法或者静态方法用来扩展，其他的功能。

示例：

函数式接口：

```java
/**
* 函数式接口，只包含一个抽象方法
*/
@FunctionalInterface
public interface MyPrint<T> {
 
    String output(T str);
 
    default void outputinfo(){
        System.out.println("info");
    }
}
```

lamdba 表达式演示：

```java
@Test
    public void fun2() {
        String strw = "aaa";
 
        // 使用匿名内部类
         System.out.println(new MyPrint<String>() {
            @Override
            public String output(String str) {
                return str+"1";
            }
        }.output(strw));
 
       // 使用lamdba 表达式
        MyPrint<String> myPrint = (str) -> str+"2";
        System.out.println(myPrint.output(strw));
    }
```

从上面演示效果，lamdba 表达式，相当于把匿名内部类中需要实现的方法实现了。那如果接口中，有两个需要被实现的方法，就不能使用lamdba 表达式，因为lamdba表达式不知道你要实现那个方法，所以只能是 函数式接口，才能用lamdba 表达式表示。

再演示一些其他例子：

排序：

对于lamdba 表达式，参数类型，return，或者花括号，在有时，都可以省略，看如下代码：

```java
List<String> strs = Arrays.asList("c", "a", "d");
// 之前的实现方式
Collections.sort(strs, new Comparator<String>() {
    @Override
    public int compare(String o1, String o2) {
        return o1.compareTo(o2);
    }
});
 
// lamdba 实现，将匿名内部类代码改为 lamdba 表达式
Collections.sort(strs, (String o1, String o2) -> {
    return o2.compareTo(o1);
});
// 只有一句，省略 return  和 花括号
Collections.sort(strs, (String o1, String o2) -> o2.compareTo(o1));
// 可以省略参数类型，会自动判断
Collections.sort(strs, (o1, o2) -> o2.compareTo(o1));
 
// ---- 上面的lamdba 表达式，idea 会提示 黄色标识，因为还可以更加简单
 
// 自然顺序的比较
Collections.sort(strs, Comparator.naturalOrder());
// 自然顺序相反的比较
Collections.sort(strs, Comparator.reverseOrder());
 
```

线程创建

```java
// 将原来  new Runnable 的代码，变成了 lamdba 表达式  
@Test
    public void fun6() {
        new Thread(
                () -> System.out.println(Thread.currentThread().getName())
        ,"thread100").start();
    }
```

### 双冒号 :: 关键字

双冒号:: 关键字，用于引用方法和构造函数

方法引用是与lambda表达式相关的一个重要特性。它提供了一种不执行方法的方法。为此，方法引用需要由兼容的函数接口组成的目标类型上下文。

oracle官方介绍：

Method References
You use lambda expressions to create anonymous methods. Sometimes, however, a lambda expression does nothing but call an existing method. In those cases, it’s often clearer to refer to the existing method by name. Method references enable you to do this; they are compact, easy-to-read lambda expressions for methods that already have a name.

方法引用

使用lambda表达式创建匿名方法。但是，有时lambda表达式只调用现有方法。在这些情况下，按名称引用现有方法通常更清楚。方法引用使您能够做到这一点；对于已经有名称的方法，它们是紧凑、易于读取的lambda表达式。

// TODO 对于方法引用，还不是很理解，但是在语义上，其实能看懂做了什么事情，有了新理解，再补充

具体示例：参考 [Java8新特性2：方法引用--深入理解双冒号::的使用](https://blog.csdn.net/aigoV/article/details/102586953)

在代码中的体现，更突显在java8 stream 中的应用

实现一个小例子：根据身份证号码，将性别自动设置到对象的性别字段中

通用的身份证号码设置性别方法

```java
// 两个参数一个是 idcard ，一个是   Consumer  接口
// 简单介绍一下 Consumer 是一个函数式接口，接受一个参数，但是不返回结果。 执行的方法 accept
private void setSexInfo(String idcard, Consumer<String> user) {
        int genderByIdCard = IdcardUtil.getGenderByIdCard(idcard); // 根据身份证，获取性别（参见 hutool 这个类库）
        if (genderByIdCard == 1) {  // 男
            user.accept("M");
        }
        user.accept("F");  // 女
    }
 
```

用户对象：

```java
public class Person{
    String sex;
   public String getSex() {
        return sex;
    }
    public void setSex(String sex) {
        this.sex = sex;
    }
}
```

测试：

```java
    @Test
    public void fun09(){
        Person person = new Person();
        // 使用匿名函数
        setSexInfo("410327188510154456", new Consumer<String>() {
            @Override
            public void accept(String s) {
                person.setSex(s);
            }
        });
        // 使用 lamdba 表达式
        setSexInfo("410327188510154456", s -> myComparator.setSex(s));
 
        // 使用 :: 双冒号
        setSexInfo("410327188510154456", person::setSex);
    }
```

对于 Consumer 接口中执行的流程，可以看做是，当  user.accept("M"); 执行时，M 会作为参数，调用对接口的实现，就是    person.setSex(s); 的逻辑

在idea 中，会给出对应的优化建议，点代码左边提示的 黄色选项 。上述代码，就可以从匿名函数到lamdba表达式到双冒号的形式的优化

![tada3F.png](https://s1.ax1x.com/2020/06/03/tada3F.png)

### Lambda的范围

参见： java8简明教程文档

对于lambdab表达式外部的变量，其访问权限的粒度与匿名对象的方式非常类似。 你能够访问局部对应的外部区域的局部final变量，以及成员变量和静态变量。

## 一些函数式接口的简单介绍

具体代码示例：参考 java8 实战 3.4.1 使用函数接口 这一章  或者 java8 简明教程中的介绍

### Predicate

接受一个输入参数，返回一个boolean 的结果。在操作stream中，filter方法中接受的参数就是 Predicate 接口

```java
    boolean test(T t);
```

示例：

```java
/**
     * Predicate 断言，一个布尔类型的函数
     * 可以用于 集合类，filter 的过滤等等
     */
    @Test
    public void fun() {
        // Predicate是一个布尔类型的函数
        Predicate<String> predicate = (s) -> s.length() > 0;
        boolean str1 = predicate.test("hello world");
        boolean str2 = predicate.test("");
 
        System.out.println(str1);
        System.out.println(str2);
 
        // 短路与 &&
        boolean test = predicate.and((s) -> s.equals("hello")).test("aaa");
        System.out.println(test);
        // 逻辑非
        boolean hello_world = predicate.negate().test("hello world");
        System.out.println(hello_world);
        // 逻辑或
        boolean aaa = predicate.or(predicate).test("aaa");
        System.out.println(aaa);
        // 判断两个对象是否相等
        boolean test2 = Predicate.isEqual("a2").test("a3");
        System.out.println(test2);
        boolean test3 = Predicate.isEqual("a3").test("a3");
        System.out.println(test3);
        boolean test4 = Predicate.isEqual(null).test("a3");
        System.out.println(test4);
 
        Predicate<String> ceshi1 = String::isEmpty;
        boolean test1 = ceshi1.test("");
        System.out.println(test1);
    }
```

 

### Function

接收一个参数，返回一个结果。可以理解为数学公式  y = f(x)，传入参数x，得到结果y.  在集合stream中，map方法接收的参数就是 Function

```java
    R apply(T t);
```

示例：

```java
/**
     * 表示接受一个参数，并返回一个结果
     * 可以理解为  y = f(x)
     * 接受x 参数，输出y，那么 f(x) 这个函数，我们自己定义就可以了
     */
    @Test
    public void fun02() {
 
        // 定义 f(x) 函数
        Function<Integer, Integer> function = (x) -> x + 1;
        // 获取结果
        Integer result = function.apply(5);
        System.out.println(result);
 
        // 定义 f(x) 函数
        Function<Integer, Integer> function1 = (x) -> x * 5;
 
        // compose 代表先执行 compose 传入的逻辑，再执行apply 的逻辑
        // 6
        Integer apply = function.compose(function1).apply(1);
        System.out.println(apply);
 
        // andThen 代表先执行当前的逻辑，再执行，andthen 传入的逻辑
        Integer apply1 = function.andThen(function1).apply(1);
        System.out.println(apply1);
 
        // ((1+1)+1)*5*5  建造者模式
        Integer apply2 = function.andThen(function1).andThen(function1).compose(function).apply(1);
        System.out.println(apply2);
 
        Function<String, Integer> toInteger = Integer::valueOf;
        Function<String, String> backToString = toInteger.andThen(String
                ::valueOf);
        backToString.apply("123"); // "123"
    }
```

 

### Supplier

不接收参数，返回一个结果。可以理解为 实体字段的get 方法

```java
    T get();
```

示例：

```java
    /**
     * 接口产生一个给定类型的结果
     */
    @Test
    public void fun03() {
        Supplier<String> str = String::new;
        String s = str.get();
        Supplier<Person> str1 = Person::new;
        // 在执行get 方法时，才会拿到person 对象 ，spring beanfactory  在执行 getbean 的时候，才会创建该对象
        // 延迟加载的功能
        Person person = str1.get();
    }
```

 

### Consumer

接收一个参数，但是不返回结果。可以理解为 实体字段的set 方法。在集合forEach 时，传入的参数就是 Consumer 接口

```java
void accept(T t);
```

示例：

```java
    /**
     * 接受一个参数输入且没有任何返回值的操作
     * 在 集合的 foreach 中 就需要填这个接口
     */
    @Test
    public void fun04() {
        Consumer<Person> personConsumer = (t) -> System.out.println("第一打印" + t.toString());
        Consumer<Person> personConsumer2 = (t) -> System.out.println("第二打印" + t.toString());
        // 执行
        personConsumer.accept(new Person());
        // 现在执行 accpect ，再执行 addthen 添加的
        personConsumer.andThen(personConsumer2).accept(new Person());
        // foreach 代码中，就调用改的 action.accept(t);
        Arrays.asList("1", "2").forEach((x) -> {
            System.out.println(x);
        });
    }
```

 

### ToIntFunction

接收一个参数，返回一个int 类型的结果，跟Function 类似，这里限定了返回的结果类型是 int

```java
    int applyAsInt(T var1);
```

 

 

 

 

 

 