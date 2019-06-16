---
title: Collectors.toMap的问题
date: 2019-06-16 19:06:47
tags:
---
#### 概述

  虽然JDK9.0已经出来了，不过我们系统最近才开始全面引入JDK1.8，JDK1.8也已经出来了好久了，各方面都挺稳定的。最近在使用lambda表达式的Collectors.toMap方法时就遇到了一个问题。
大致源码如下：

```
public class Test {
    public static void main(String[] args) {
        // initMemberList为获取数据的方法
        List<Member> list = Test.initMemberList();
        Map<String, String> memberMap = list.stream().collect(Collectors.toMap(Member::getId, Member::getImgPath));
        System.out.println(memberMap);
    }
}

class Member {
    private String id;
    private String imgPath;

    // get set省略
}
```
运行程序，直接提示：
```
Exception in thread "main" java.lang.NullPointerException
    at java.util.HashMap.merge(HashMap.java:1224)
    at java.util.stream.Collectors.lambda$toMap$58(Collectors.java:1320)
    at java.util.stream.ReduceOps$3ReducingSink.accept(ReduceOps.java:169)
    at java.util.ArrayList$ArrayListSpliterator.forEachRemaining(ArrayList.java:1374)
    at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:481)
    at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:471)
    at java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:708)
    at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234)
    at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
    at com.jdk.test.Test.main(Test.java:13)
```
 想来想去，一开始一直没想明白为什么会为空指针呢，因为list是不可能为null的，无奈后来拿着java.util.HashMap.merge加NullPointerException异常上网搜索了一下。原来是由于在由对象转为Map的时候，转为map的value是null导致的。大概如下：
```
public static List<Member> initMemberList() {
        Member member1 = new Member();
        member1.setId("id_1");
        member1.setImgPath("http://www.baidu.com");
        // 这里有一个null导致的
        Member member2 = new Member();
        member2.setId("id_2");
        member2.setImgPath(null);

        List<Member> list = new ArrayList<>();
        list.add(member1);
        list.add(member2);
        return list;
    }
```
大致看了下源码，原来Collectors.toMap底层是基于Map.merge方法来实现的，而merge中value是不能为null的，如果为null，就会抛出空指针异常，原来问题是这样的。
```
Collectors.toMap() internally uses Map.merge() to add mappings to the map. Map.merge() is spec'd not to allow null values, regardless of whether the underlying Map supports null values. This could probably use some clarification in the Collectors.toMap() specifications.
```
看了下，在openJDK的bug列表里还有这个呢：JDK-8148463，不知道这到底算不算bug呢。地址：[Collectors.toMap fails on null values](https://bugs.openjdk.java.net/browse/JDK-8148463)

问题归问题，我们还是需要通过其他的方式解决的。

##### **解决方式1**

原来for循环的方式，亦或是forEach的方式：

```
Map<String, String> memberMap = new HashMap<>();
list.forEach((answer) -> memberMap.put(answer.getId(), answer.getImgPath()));
System.out.println(memberMap);

Map<String, String> memberMap = new HashMap<>();
for (Member member : list) {
    memberMap.put(member.getId(), member.getImgPath());
}

```

##### **解决方式2**

使用stream的collect的重载方法：

```
Map<String, String> memberMap = list.stream().collect(HashMap::new, (m,v)->
    m.put(v.getId(), v.getImgPath()),HashMap::putAll);
System.out.println(memberMap);

```

##### **解决方式3**

```
> 继承Collector，手动实现toMap方法，然后调用我们自己封装的toMap方法就可以了。有关实现Collector，可参考：[JDK8 Stream API中Collectors中toMap方法的问题以及解决方案](http://blog.jobbole.com/104067/)
```

  其实不管这是不是bug，说到底，还是JDK1.8的lambda表达式用的太少，了解的太少导致的问题。所以说还是应该多去使用新技术，多踩坑。

stackoverflow地址：[#](https://stackoverflow.com/questions/24630963/java-8-nullpointerexception-in-collectors-tomap) [Java 8 NullPointerException in Collectors.toMap](https://stackoverflow.com/questions/24630963/java-8-nullpointerexception-in-collectors-tomap)

##### 后记

使用Collector.toMap又发现了一个问题，Map中的key不能重复，如果重复的话，会抛出异常：

```
public static List<Member> initMemberList() {
    Member member1 = new Member();
    member1.setId("id_1");
    member1.setImgPath("http://www.google.com");

    Member member2 = new Member();
    member2.setId("id_1");
    member2.setImgPath("http://www.baidu.com");

    List<Member> list = new ArrayList<>();
    list.add(member1);
    list.add(member2);
    return list;
}

```

以上代码，运行时，提示错误：

```
Exception in thread "main" java.lang.IllegalStateException: Duplicate key http://www.google.com
    at java.util.stream.Collectors.lambda$throwingMerger$0(Collectors.java:133)
    at java.util.HashMap.merge(HashMap.java:1253)
    at java.util.stream.Collectors.lambda$toMap$58(Collectors.java:1320)
    at java.util.stream.ReduceOps$3ReducingSink.accept(ReduceOps.java:169)
    at java.util.ArrayList$ArrayListSpliterator.forEachRemaining(ArrayList.java:1374)
    at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:481)
    at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:471)
    at java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:708)
    at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234)
    at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
    at com.jdk.test.Test.main(Test.java:13)

```

通过查看Collectors.toMap的代码及注释我们会发现：

```
public static <T, K, U> Collector<T, ?, Map<K,U>> toMap(Function<? super T, ? extends K> keyMapper,
                                Function<? super T, ? extends U> valueMapper) {
    return toMap(keyMapper, valueMapper, throwingMerger(), HashMap::new);
}

```

```
> If the mapped keys contains duplicates (according to Object#equals(Object)), an IllegalStateException is thrown when the collection operation is performed. If the mapped keys may have duplicates, use toMap(Function, Function, BinaryOperator) instead.
```

  所以说呢，我们使用的Collectors.toMap的方法是不支持key重复的，并且如果有重复的时候，建议我们使用toMap(Function, Function, BinaryOperator) 方法来替换使用，并且我们还可以定义当key重复的时候，是使用旧的数据还是使用新的数据呢，除了选择使用新旧数据，当然也可以做一些额外的操作，**但该方法还是会有value为null的问题哦**。
即：

```
Map<String, String> memberMap = list.stream().collect(Collectors.toMap(Member::getId, Member::getImgPath,
(oldValue,newValue) -> oldValue));
System.out.println(memberMap);

```

同样，在openJDK的bug列表里自然也少不了这个小bug：JDK-8040892
[Incorrect message in Exception thrown by Collectors.toMap(Function,Function)](https://bugs.openjdk.java.net/browse/JDK-8040892)

不过，在JDK9里这个bug应该是被修复了的：JDK-8173464
[Wrong exception message when collecting a stream to a map](https://bugs.openjdk.java.net/browse/JDK-8173464)

> Pallavi Sonal added a comment - 2017-01-27 00:42
> This has been fixed in JDK 9 with JDK-8040892.
> In JDK8 versions, it throws the wrong message i.e. instead of Duplicate key <KEY>, it shows Duplicate key <VALUE>.

  再多说一句，toMap方法还有一个重载方法，是可以指定一个Map的具体实现，该方法或许有时候我们会用到呢。

```
Map<String, String> memberMap = list.stream().collect(Collectors.toMap(Member::getId, Member::getImgPath,
    (oldValue,newValue) -> oldValue, HashMap::new));
System.out.println(memberMap);
```

