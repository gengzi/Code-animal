[TOC]

## 目标

* 熟悉stream 大部分的操作

  参考： java8 实战书籍（推荐阅读） 、java8 简明教程

## 示例

基本会把Stream 类中的方法都实验一遍，具体的方法信息可以参考 java8 的中文api文档

具体的使用，也可以搜索 java8 简明教程 ，中的使用方法

以下的代码可以参考：

fun.gengzi.codecopy.java8.lamdba.StreamTest

fun.gengzi.codecopy.java8.lamdba.utils.MyTest

使用菜肴实体来演示 相关api

```java
public class Dish {
    private final String name;
    private final boolean vegetarian;
    private final int calories;
    private final Type type;
 
    public Dish(String name, boolean vegetarian, int calories, Type type) {
        this.name = name;
        this.vegetarian = vegetarian;
        this.calories = calories;
        this.type = type;
    }
    public String getName() {
        return name;
    }
    public boolean isVegetarian() {
        return vegetarian;
    }
    public int getCalories() {
        return calories;
    }
    public Type getType() {
        return type;
    }
    @Override
    public String toString() {
        return name;
    }
    public enum Type {MEAT, FISH, OTHER}
}
```

数据准备：

```java
List<Dish> menu = Arrays.asList(
            new Dish("pork", false, 800, Dish.Type.MEAT),
            new Dish("beef", false, 700, Dish.Type.MEAT),
            new Dish("chicken", false, 400, Dish.Type.MEAT),
            new Dish("french fries", true, 530, Dish.Type.OTHER),
            new Dish("rice", true, 350, Dish.Type.OTHER),
            new Dish("season fruit", true, 120, Dish.Type.OTHER),
            new Dish("pizza", true, 550, Dish.Type.OTHER),
            new Dish("prawns", false, 300, Dish.Type.FISH),
            new Dish("prawns", false, 300, Dish.Type.FISH),
            new Dish("salmon", false, 450, Dish.Type.FISH));
```

 

### 创建流

Stream.of()

集合.stream()

Arrays.stream(数组)

Map.values/entrySet/keySet.stream()

```java
  @Test
    public void fun() {
        Arrays.asList("1", "2")
                .stream()
                .findFirst()
                .ifPresent(s -> System.out.println(s));
        // 使用 stream 创建一个流
        Stream.of("1", "22", "33")
                .findFirst()
                .ifPresent(s -> System.out.println(s));
        // 通过jdk 提供的 流来操作
        // IntStream
        // LongStream
        IntStream.range(1, 4)
                .forEach(s -> System.out.print(s));
        // range(1,4) 提供一个从 1到 3 的数据范围，如果了解python
        // 会发现，这个跟 python 提供的 range() 函数类似
    }
 
```

 

### filter 过滤数据

filter(返回一个boolean) ,使用函数式接口是 Predicate ，返回一个 boolean ，参数是 T

```java
// 查询出 热量高于350 的菜肴
Stream<Dish> dishStream = menu.stream()
                .filter(n -> n.getCalories() > 350);
    /**
     * 查询出 热量高于350 的菜肴名称,从大到小排序,只获取前三个
     */
    @Test
    public void fun01() {
 
        List<String> menuNames = menu.stream()
                .filter(menuinfo -> menuinfo.getCalories() > 350)
                .sorted(Comparator.comparing(menuinfo -> menuinfo.getCalories()))
                .limit(3)   // 该方法会返回一个不超过给定长度的流
                .map(menuinfo -> menuinfo.getName())
                .collect(Collectors.toList());
 
        menuNames.forEach(menuName -> {
            System.out.println(menuName);
        });
 
        Stream<Dish> dishStream = menu.stream()
                .filter(n -> n.getCalories() > 350);
 
    }
```

 

### shortd 排序

shortd() 默认自然排序

stortd(Comparator.comparing(xxx)) 自定义排序

其实理解 stortd(Comparator.comparing(xxx)) 的参数 ， 直接理解为语义，比如根据 热量高低排序，就直接写 Dish::getCalories ，如果取反序， Comparator.comparing(Dish::getCalories).reversed() 加一个 reversed 就是反序，像 stream 提供的这种链式方法调用，将我们想做的事情语义话，比如 我想要先过滤热量高于350 的菜肴，再将这些菜肴根据热量高低排序，对于Stream 来说，就是 .filter .sorted 就完事了，非常容易理解和使用。

这里可以使用 System.out.println 去打印每次执行的数据，会发现像一条数据流一样，每个数据从第一个方法流转到其他方法。一个数据完成接着执行第二个数据，依然从第一个方法开始流转，直到所有数据被执行完毕。

```java
  @Test
    public void fun07() {
        List<Dish> collect1 = menu.stream()
                .filter(info -> {
                    System.out.println("filter:" + info.getName());
                    return info.getType().equals(Dish.Type.MEAT);
                })
                .map(info -> {
                    System.out.println("map:" + info.getName());
                    return new Dish(info.getName(), true, info.getCalories(), info.getType());
 
                })
                .sorted(Comparator.comparing(Dish::getCalories))
                .collect(Collectors.toList());
 
        collect1.forEach(info -> System.err.println(info.getName()+":"+info.getCalories()));
 
 
    }
```

 

### distinct 去重

distinct() 去重，跟sql 语句中提供的 distinct 类似，注意一般去重都是对指定基本类型或者字符串类型，如果去重是对象，应该比较的是对象地址。

```java
   /**
     * 筛选出 不重复的菜单信息
     */
    @Test
    public void fun02() {
        List<String> menuNames = menu.stream()
                .map(Dish::getName)
                .distinct()
                .collect(Collectors.toList());
        menuNames.forEach(menuName -> {
            System.out.println(menuName);
        });
 
    }
```

 

### collect 聚合数据

collect(Collectors.toList())

collect(Collectors.toSet())

collect 应该是最经常用到的方法，因为Stream 执行的方法，不会去改变底层的数据，如果你要拿到处理过的数据，就用 collect 方法将数据封装到一个集合中返回。

collect 可以将数据聚合为 list  set  map 等等，经常使用的就是 toList toSet  toMap

当然比如 Collectors.groupingBy     Collectors.joining 可能会用到

```java
        List<String> strings = Arrays.asList("是", "否");
        Set<List<String>> collect1 = Stream.of(strings).collect(Collectors.toSet());
 
        //返回顺序排列流，其元素为指定的值
        List<String> collect = Stream.of("是", "否").collect(Collectors.toList());
 
        // groupingBy 返回  Map<K, List<T>>, k 就是分组的条件，值是通过 tolist 转化的 list 对象
        Map<Integer, List<Transaction>> collect = transactions.stream()
                .collect(Collectors.groupingBy(transaction -> transaction.getYear()));
 
 
    @Test
    public void fun100() {
        List<Person> list = new ArrayList();
        list.add(new Person(1, "haha"));
        list.add(new Person(2, "rere"));
        list.add(new Person(3, "fefe"));
//Map<Integer, Person> mapp = list.stream().collect(Collectors.toMap(Person::getId, Function.identity()));
//Map<Integer, Person> mapp = list.stream().collect(Collectors.toMap(x -> x.getId(), x->x));
//System.out.println(mapp);
        Map<Integer, String> map = list.stream().collect(Collectors.toMap(Person::getId, Person::getName));
//Map<Integer, String> map = list.stream().collect(Collectors.toMap(Person::getId, Person::getName,(x1,x2)->x1))
 
    }
```

 

### map 映射字段

map(字段) ，一般映射对象中某个字段

```java
    /**
     * 查询出菜单名中，包含的所有字符，去重
     */
    @Test
    public void fun04() {
 
        List<String[]> collect = menu.stream()
                .map(menuinfo -> menuinfo.getName().split(""))  // 映射某一列
                .distinct()
                .collect(Collectors.toList());
    }
```

### limit 取前几个

limit(数字),取前几个数据，有点类似mysql 的limit ，可以与 skip 搭配使用

```java
    /**
     * 查询出 热量高于350 的菜肴名称,从大到小排序,只获取第三个
     */
    @Test
    public void fun03() {
 
        List<String> menuNames = menu.stream()
                .filter(menuinfo -> menuinfo.getCalories() > 350)
                .sorted(Comparator.comparing(menuinfo -> menuinfo.getCalories()))
                .limit(3)
                .skip(2)  // 返回一个扔掉了前 n 个元素的流  如果流中元素不足 n 个，则返回一 个空流
                .map(menuinfo -> menuinfo.getName())  // 映射某一列
                .collect(Collectors.toList());
 
 
        menuNames.forEach(menuName -> {
            System.out.println(menuName);
        });
    }
```

### skip 跳过几个

skip(数字)

参见上面的代码

### xxMath

注意判断能否匹配到，是一个终止操作。所以不要使用同一个 stream 执行多次的 xxMath 操作，会报异常的，不信你试试。

### anyMatch 包含任意一个

anyMatch() 只要有一个，返回 true

```java
    @Test
    public void fun07() {
        List<String> stringCollection = new ArrayList<>();
        stringCollection.add("ddd2");
        stringCollection.add("aaa2");
        stringCollection.add("bbb1");
        stringCollection.add("aaa1");
        stringCollection.add("bbb3");
        stringCollection.add("ccc");
        stringCollection.add("bbb2");
        stringCollection.add("ddd1");
        // sorted只是创建一个流对象排序的视图,不会改变原来集合的顺序
        stringCollection.stream()
                .sorted(Comparator.naturalOrder())
                .filter(s -> s.startsWith("b"))
                .forEach(s -> System.out.println(s));
 
        // map 它能够把流对象中的每一个元素对应到另外一个对象上。
        stringCollection.stream()
                .sorted()
                .map(String::toUpperCase)
                .filter(s -> s.startsWith("A"))
                .forEach(s -> System.out.println(s));
 
 
        // match 判断某一种规则是否与流对象相互吻合的 终结操作
 
        // 检测所有都是 a 开头的
        boolean a = stringCollection.stream()
                .allMatch(s -> s.startsWith("a"));
        System.out.println(a);
 
        // 检测没有 a 开头的
        boolean a1 = stringCollection.stream()
                .noneMatch(s -> s.startsWith("a"));
        System.out.println(a1);
 
        // 检测可能 a 开头的
        boolean a2 = stringCollection.stream()
                .anyMatch(s -> s.startsWith("a"));
        System.out.println(a2);
        }
```

### allMatch 包含所有

allMatch()  所有都包含，才返回 true

代码参见上面

### noneMatch 不包含任意一个

noneMatc()  不包含任意一个，才返回true

代码参见上面

### flatmap 将多个流合并为一个流

flatmap() 这个和map() 方法的区别是，flatmap 可以将一个流中的每个值都转为另一个流，并将所有的流连接起来成为一个流。也就是使用 map 映射出来类似是 Stream<Stream<xxx>>

不好处理这种流，就可以使用 flatmap 映射为 Stream<xxx> ，将流进行合并

```java
// 使用 groupingBy 将其分组      
Map<Integer, List<Transaction>> collect = transactions.stream()
                .collect(Collectors.groupingBy(transaction -> transaction.getYear()));
// map.values() 返回的是一个值的集合，使用stream操作这个集合
Stream<List<Transaction>> stream = collect.values().stream();
// 使用map 方法
Stream<Stream<Transaction>> streamStream = stream.map(transactions1 -> transactions1.stream());
// 使用 flatMap 方法   
Stream<Transaction> transactionStream = stream.flatMap(transactions1 -> transactions1.stream());
```

### reduce 合并数据

reduce 常用来 求和，求最大值，最小值

```java
  Integer reduce = persons.stream()
                .map(person -> person.getId())
                // 第一个参数 初始化的值  第二个参数 BinaryOperator 将两个元素结合起来变成一个新值
                // 在执行流中， p1 作为计算的第一个参数 0 ，从流中获取数据，作为 p2 的参数值，进行计算，然后再从流中获取数据，进行累加
                .reduce(0, (p1, p2) -> {
                    return p1 + p2;
                });
        System.out.println(reduce);
 
        Person person = persons.stream()
                // p1 是 new person 的第一个对象，p2 是从流中获取的
                // 关于 id 的累加，结果是 76 ，我们把 id 改为 1 ，结果应该变成 77
                .reduce(new Person(1, "person"), (p1, p2) -> {
                    p1.setId(p1.getId() + p2.getId());
                    p1.setName(p1.getName() + p2.getName());
                    return p1;
                });
        System.out.println(person.toString());
 
 
        Integer reduce1 = persons.stream()
                .map(person1 -> person1.getId())
                // Integer 的 内部，完成了 retur a+b 的操作
                .reduce(0, Integer::sum);
 
        Optional<Integer> reduce2 = persons.stream()
                .map(person1 -> person1.getId())
                // 计算最大值，因为没有提供初始化的参数，所以返回了一个 Optional 对象
                .reduce(Integer::max);
        // 如果存在
        reduce2.ifPresent(integer -> System.out.println(integer));
        Integer integer1 = reduce2.filter(integer -> integer > 0).orElse(0);
        System.out.println(integer1);
 
        persons.stream()
                //三个参数，第一个是初始化参数，第二个参数 每次都会拿 初始化参数 0 进行相加，第三个参数，会把得到的 sum 值，进行相加
                .reduce(0, (sum, person1) -> {
                    sum += person1.getId();
                    return sum;
                }, (sum1, sum2) -> {
                    return sum1 + sum2;
                });
```

 

 