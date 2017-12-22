---
title: 使用Optional来减少null检查
date: 2017-12-21 17:57:44
layout: tag
tags: [Java, Optional]
---


## 由来

平常我们使用null检查在项目中简直太常见了，从数据库中查询到的数据可能不存在返回null，service中处理中发现不存在返回一个null，在互相调用的时候每次都需要做`(if obj != null)`的判断，散落在程序中很难看。更难看的是当你遗漏了一个空指针判断，程序就会无情的给你抛出一个NPE让你知道谁才是老大。

假设我们有一个用户类User，用户可以有收货地址类Addr，收货地址中肯定会有省province属性啦，如果我们要获取用户的收货省，简单的来一个链式操作`user.getAddr().getProvince()`,这样操作可以么?

<!--more-->

## 以往的null检查方式

用户在新注册之后可能是没有收货地址的，因此`user.getAddr()`返回null，再调用就会给你点颜色看看。因此我们必须要先判断null，然后才能决定是否可以调用`getProvince`方法
```java
public String getUserConsigneeProvince(User user){
  if (user != null){
     Addr addr = user.getAddr();
     if (addr != null){
       return addr.getProvince();
     }
  }
  return null;  
}
```

或者使用防御式编程方式(以前我喜欢的编程方式),在检查到null后直接处理

```java
public String getUserConsigneeProvince(User user){
  if (user == null){
    return null;
  }
  Addr addr = user.getAddr();
  if (addr == null){
    return null;
  }
  return addr.getProvince();
}
```


在这里我们也很鸡贼的返回了一个null来表示用户的收货省不存在,给以后的使用这个方法的人(当然也包括自己)挖了一个坑，如果直接返回给前端，那页面上就会有一个大大的null等着QA给你提bug吧。


## 1.8中对Null的处理

在Haskell中有一个Maybe类来处理可能的null，Scala中也提供了Option[T]来表示，Kotlin中使用在调用后加?来安全的处理返回值为null的情况。Java1.8借鉴了Haskell和Scala中方式，提供了一个Optional类来帮助程序员避免null检查。


### 设计哲学

我们看到在获取收货省的api中返回值直接是一个String，我们是不可能从这个返回值上面看出用户的收货省是否存在的，因此在设计时，对于可能不存在的值，我们选择返回一个Optional来表示你需要处理不存在的情况。

因此我们把返回值改写成`Optional<String>`
```java
public Optional<String> getUserConsigneeProvince(User user){
  if (user == null){
    return Optional.empty();
  }
  Addr addr = user.getAddr();
  if (addr == null){
    return Optional.empty();
  }
  return Optional.of(addr.getProvince());
}
```

不过这并没有改善我们的内部处理，还是判断了null，因此我们先来看一下Optional的API，来来看一下用Optional如何简化我们的书写方式。

### 处理方式

#### API

Optional<T> Api:

| 名称 | 返回值 | 参数 | 说明|
| ------- | ------- | ------: | :------: |
|isPresent | boolean | void | 如果不存在值则false，存在为true|
|ifPresent | void | Consumer| 如果存在则调用Consumer消费值 |
|map | Function | Optional<U> | 对值做映射 |
|flatMap | Function | Optional<U> | 对值做扁平映射 | 
|orElse | T | T | 存在返回包含的值，不存在就返回这里面的值 |
|orElseGet | T | Supplier<T> | 返回用supplier 生成的一个值 |
|filter | Predicate | Optional<T> | 过滤值 |
| get| T| void | 获取包装的值,不存在会抛出异常|


可以看到API设计中使用到了函数式相关的东西，使得我们调用的时候可以使用lambda或者行为参数化的方式更方便的使用
在map和flatMap等API中隐含了null的判断，使得我们不用在应用中显式的去做null判断了。

#### 一个参数的实践

接下来就是见证奇迹的时刻：
```java
public Optional<String> getUserConsigneeProvince(User user){
  return Optional.ofNullable(user).map(User::getAddr).map(Addr::getProvince);
}
```
我们使用1行代码代替了6行，而且表达的更加清晰

当然如果这个API很多人使用，很难改变返回值的话我们可以使用orElse做值处理，如下：
```java
public String getUserConsigneeProvince(User user){
  return Optional.ofNullable(user).map(User::getAddr).map(Addr::getProvince).orElse("");
}
```
get方法只有我们在确定值一定存在的情况下才能使用。

可以看到我们并未改变接口参数和返回值，但在内部处理上经过重写我们已经简化了非常多的代码，逻辑也变得清晰，更具有可读性。

为何可以如此写呢？Optional类其实是将null判断内化了，将null判断从用户手中接过来变成自己API的一部分，把用户从null判断的深渊中解放出来，只用关注自己的业务处理逻辑。

#### 两个参数的处理
上面是一个参数的处理，如果我们有两个参数该怎么办呢。

假设有两个用户，我们需要判断一下他们是否是同一个省的，就使用上面已经提供的获取省的方法
```java
public static boolean isSameProvince(User user1, User user2) {
    String user1Province = getUserConsigneeProvince(user1);
    String user2Province = getUserConsigneeProvince(user2);
    if (user1Province == null || user2Province == null) {
        return false;
    }
    if (user2Province.equals(user2Province)) {
        return true;
    }
    return false;
}
```
还好还好，只有一个if判断了null的情况，可是我们也用了9行才完成了这个简单的功能，我们其实最需要的equals这个判断，上面的三行null相关应该是需要避免的。
我们用Optional来改写一下，来感受下威力。

```java
*public static boolean isSameProvince2(User user1, User user2) {
        return Optional.ofNullable(getUserConsigneeProvince(user1))
            .map(p -> Optional.ofNullable(getUserConsigneeProvince(user2))
                .filter(p2 -> p.equals(p2)))  
            .isPresent();
}
```
为了可读性我们写了4行，就算如此我们也减少了一半的代码，而且逻辑上更连贯，没有打断我们的可恶的null了。

## 总结

如上可以看出Optional在使用上带给我们的变化，让我们可以摆脱以往的null，用更加健康的调用方式来编写。也增加代码的可读性，逻辑上一气呵成。希望大家在平常多多使用。尽快远离恼人的null。
