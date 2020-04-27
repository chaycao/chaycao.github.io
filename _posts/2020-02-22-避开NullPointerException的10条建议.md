---
layout:     post
title:      避开NullPointerException的10条建议
date:       2020-02-22
author:     ChayCao
header-img: img/post-bg-2015.jpg 
catalog: true
tags:  Java
---


## 1. 引言

`NullPointerException`应该是 Java 开发中最常出现的问题，也是 Java 程序员最容易犯的错误。虽然看起来是个小错误，但带来的影响却不小，Tony Hoare（null 引用的发明者）在 2009 年说过 NPE 大约给企业造成数十亿美元的损失。在这工作半年内，我就踩了好几次 NPE 的坑。举个例子，我需要在原有逻辑上加一段代码，而新加的代码报错抛出了 NPE，同时又没做异常处理，就直接导致后面的逻辑不运行了，影响了整个原有逻辑，太恐怖了。所以大家一定要小心避开 NPE 这个坑。

本文将会从以下两个方面说起：

1. 发生 NPE 的可能情况
2. 避开 NPE 的建议



## 2. 发生 NPE 的可能情况

首先我们需要清楚 NPE 是怎么发生的。

```java
String s;
String[] ss;
```

当声明一个引用变量时，若未指定其指向的内容，Java 会将其默认指向 null，一个空地址，意味着“什么都没有指向”。后续若也没有为该变量赋值，则当使用这个变量里的内容时，便会抛出 NPE。

例如通过`.`去访问方法或者变量，`[]`去访问数组插槽：

```java
System.out.println(s.length());
System.out.println(ss[0]);
```

以下是 **NPE 的 Javadoc 概述的 6 个可能发生情况**：

1. **在空对象上调用实例方法**。对空对象调用静态方法或类方法时，不会报 NPE，因为静态方法不需要实例来调用任何方法；

2. **访问或更改空对象上的任何变量或字段时**；
3. **抛出异常时抛出 null**；
4. **数组为 null 时，访问数组长度**；
5. **数组为 null 时，访问或更改数组的插槽**；
6. **对空对象进行同步或在同步块内使用 null**。



## 3. 避开 NPE 的建议

这节将介绍如何在开发过程中避开 NPE 的一些建议。

（1）**尽量避免在未知对象上调用 equals() 方法和 equalsIgnoreCase() 方法，而是在已知的字符串常量上调用**

由于 `equals()` 和 `equalsIgnoreCase()` 具有对称性，所以可以直接翻转，这是很容易实现的。

```java
Object unknowObject = null;
if (unknowObject.equals("knowObject")) {
    System.out.println("如果 unknowObject 是 null，则会抛出 NPE");
}
if ("knowObject".equals(unknowObject)) {
    System.out.println("避免 NPE");
}
```



（2）**避免使用 toString()，而是 String.valueOf()**

这是因为 `String.valueOf()` 中做了非空校验，同样里面也调用了对象的 `toString()`方法，所以结果是相同的。

```java
Object unknowObject = null;
System.out.println(unknowObject.toString());
System.out.println(String.valueOf(unknowObject));
```



（3）**使用 null 安全的方法和库**

开源库的方法通常都了非空校验，例如 Apache common 库中的 `StringUtils` 工具类中的 `isBlank()`、`isNumeric()` 等方法，使用时不必担心 NPE。那我们在使用第三方库时，一定要了解它是否是 null 安全的，如果不是，则需要我们自己做好非空校验。

```java
System.out.println(StringUtils.isBlank(null));
System.out.println(StringUtils.isNumeric(null));
```



（4）**当方法返回集合或数组时，避免返回 null，而应是空集合或空数组**

返回空集合或空数组时，可以保证调用方法（如`size()`、`length()`）不会出现 NPE。而且`Collections` 类中提供了方便的空 List、Set和Map，`Collections.EMPTY_LIST`、`Collections.EMPTY_Set`、`Collections.EMPTY_MAP`。

```java
public List fun(Customer customer){
   List result = Collections.EMPTY_LIST;
   return result;
}
```



（5）**使用 @NotNull 和 @Nullable 注解**

- `@NonNull`可以标注在方法、字段、参数之上，表示对应的值不可以为空
- `@Nullable`可以标注在方法、字段、参数之上，表示对应的值可以为空

以上两个注解在程序运行的过程中不会起任何作用，只会在IDE、编译器、FindBugs检查、生成文档的时候提示。

有好几种 `@NotNull` 和 `@Nullable`，我还没能搞明白，具体怎么使用我先不讲了。但即使不谈检测，单纯作为标识也是能够起到文档的作用。



（6）**避免不必要的装箱拆箱**

如果包装对象为 null，在拆箱时容易发生 NPE。

```java
Integer integer = null;
int i = integer;
System.out.println(i);
```



（7）**定义合理的默认值**

定义成员变量时提供合理的默认值。

```java
public class Main {
    private List<String> list = new ArrayList<>();
    private String s = "";
}
```



（8）**使用空对象模式**

空对象是设计的一种特殊实例，为方法提供默认的行为，例如 `Collections`中的 `EMPTY_List`，我们仍能使用它的 `size()`，会返回 0，而不会抛出 NPE。

再举个 Jackson 中的例子，当子节点不存在时，`path()`会返回一个 `MissingNode` 对象，当调用 `MissingNode` 对象的 `path()` 方法是将继续返回 `MissingNode`。这样的链式调用将不会抛出 NPE。最后返回后，用户只需检查结果是否为 `MissingNode` 就能判断是不是找到了。

```java
JsonNode child = root.path("a").path("b");
if (child.isMissingNode()) {
    //...
}
```



（9）**Optional**

`Optional` 是 Java8 的一个新特性，可以为 null 的容器对象。若值存在，不为 null，则 `isPresent()`方法会返回 true，调用 `get()`方法可返回该对象。它所起到的作用是避免我们显示的进行空值校验。

举一个常见的空值校验示例：

```java
// 最外层
public class Outer {
    Nested nested;
    Nested getNested() {
        return nested;
    }
}
```

```java
// 第二层
public class Nested {
    Inner inner;
    Inner getInner() {
        return inner;
    }
}
```

```java
// 最底层
public class Inner {
    String foo;
    String getFoo() {
        return foo;
    }
}
```

我们通过 `Outer` 对象访问 `Inner` 中的 `foo` 属性，若加空值校验的话，代码如下：

```java
Outer outer = new Outer();
if (outer != null) {
    if (outer.nested != null) {
        if (outer.nested.inner != null) {
            System.out.println(outer.nested.inner.foo);
        }
    }
}
```

这种嵌套式的判断语句在空值校验中很常见。而使用 `Optional` 再结合 Java8 的特性 Lambda 表达式、流处理，可以采用链式操作，更为简洁。

```java
Optional.of(new Outer())
    .map(Outer::getNested)
    .map(Nested::getInner)
    .map(Inner::getFoo)
    .ifPresent(System.out::println);
```

`Optional.of()` 方法可以返回一个 `Optional<Outer>` 的对象，并将 `Outer` 对象放在容器内，`Optinal.map()`方法中，会通过 `isPresent()` 方法判断是否为 null，如果为 null，将返回 `Optional<Outer>` 类型的空对象，不影响后续的链式调用。是不是很眼熟，这和我们在第 8 点说的空对象模式类似，在 `Optional` 的实现中也采用了这种模式。



（10）**细心**

嘿嘿，凑个第十点吧。

最后祝大家成功避开 `NullPointerException`，有什么其他的好建议，欢迎留言交流！



## 4. 参考

1. [Java Tips and Best practices to avoid NullPointerException in Java Applications](https://javarevisited.blogspot.com/2013/05/ava-tips-and-best-practices-to-avoid-nullpointerexception-program-application.html )
2. [如何在 Java8 中风骚走位避开空指针异常](<https://juejin.im/post/5c41d8ae6fb9a049a42f575b#comment>)


![公众号二维码](http://i2.tiimg.com/717558/a410997819862ca9.png)
