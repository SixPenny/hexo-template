---
title: Java动态代理与Cglib代理
date: 2017-12-21 18:27:44
layout: tag
tags: [Ansible, Cglib, 代理]
---

最近又继续回来死磕Spring源码，以前看的也忘得差不多了，这次先把Spring使用的动态代理cglib看了一下，打好基础知识。
cglib使用上特别简单，而且也不像Java要实现动态代理一样必须有接口，看一下cglib的wiki可以很容易上手。

<!--more-->

## 实现前的准备
我们先准备测试用到的类和接口，简单的写一个test,假设是我们平常写的简单的dao
```java
public interface TestDao {
    public String test();
}
```

然后写一个实现
```java
public class TestDaoImpl implements TestDao {

    public String test() {
        System.out.println("test dao impl");
        return "test";
    }
}

```

里面就是简单的crud操作，现在如果我们需要对dao开启事务控制，我们当然可以直接在dao实现类中来做这个操作，不过对代码的侵入性很强，需要硬编码到Dao类中，而且重复代码会分布到每个类中。如果用代理来实现，那就会很优雅完美

## Java动态代理的实现

首先来定义代理要实现的功能
```java
public class Aop implements InvocationHandler {
    Object target;
    public Aop(Object o){
        this.target = o;
    }
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("java dynamic before");
        Object re = method.invoke(target, args);
        System.out.println("java dynamic after");
        return re;
    }
}
```

然后写一个生成代理类的工厂：
```java
public class JavaDynamicObjectFactory {

    public static <T> T getProxiedObject(Class clazz){

        Aop aop = null;
        try {
            aop = new Aop(clazz.newInstance());
            T proxied = (T) Proxy.newProxyInstance(JavaDynamicObjectFactory.class.getClassLoader(), clazz.getInterfaces(), aop);
            return proxied;
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return null;
    }

    public static void main(String[] args) {
        TestDao testDao = JavaDynamicObjectFactory.getProxiedObject(TestDaoImpl.class);
        testDao.test();
    }
}

```
测试后输出：
> java dynamic before
> test dao impl
> java dynamic after

## Cglib代理实现

cglib也需要实现一个接口
```java
public class Aop implements MethodInterceptor {
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("cglib before");
        Object re = methodProxy.invokeSuper(o, objects);
        System.out.println("cglib after");
        return re;
    }
}
```

实现cglib工厂

```java
public class CglibObjectFactory {

    public static TestDao getTestService(){
        return new TestDaoImpl();
    }

    public static <T> T getProxiedObject(Class clazz){
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(new Aop());
        T proxied = (T) enhancer.create();

        return proxied;
    }

    public static void main(String[] args) {
        TestDao testDao = CglibObjectFactory.getProxiedObject(TestDaoImpl.class);
        testDao.test();
    }
}

```

执行后输出：
> cglib before
> test dao impl
> cglib after

当然cglib不仅仅这点功能，还提供了Bean generator ,Bean copier,Bean map等工具类功能，不过核心还是代码生成的

## 总结对比

cglib是直接操作字节码生成的代理类，底层依赖了ASM，Java的dynamic是在运行期增强，而且速度也一直受人诟病，平常如果有需要的话使用cglib还是很不错的，简单易上手。

## 废话几句

昨天在stackoverflow上看到一个关于代理框架的讨论，发现cglib有很多问题，很长时间没有更新，现在放到了GitHub上，然而更新解决问题依然很慢，不建议使用了。大家可以去尝试一下Javaassist，ASM等框架。ASM以前自己折腾过，不过看到后面全是byte code头晕，就放弃了，后面找机会再入坑


## 参考资料

[github上的cglib tutorial](https://github.com/cglib/cglib/wiki/Tutorial)
[stackoverflow上的讨论Are there alternatives to cglib?](https://stackoverflow.com/questions/2261947/are-there-alternatives-to-cglib)