---
title: "一些我认为有用有趣的JDK方法"
description: "介绍一些我认为有用有趣的JDK方法如java.util.Objectsjava.lang.System、sun.reflect.Reflection以及java.util.Base64等"
publishDate: "09 Apr 2018"
tags: ["JDK", "java"]
---

在学习JDK的源码过程中我遇到了一些有趣有用的方法，在此之前如果要使用这些工具方法，我首先会想到的是`commons-lang`和`guava`这样的*语言扩展包*，但现在如果是写一些demo，使用原生即可达到目的。当然我们也不能否认它们的作用，在平时的工作项目中几乎都会引入这些*语言扩展包*，直接使用他们也使得编程风格统一，而且还能够对低版本的JDK提供支持。
以下收集的代码片段可能会逐渐增加，也可能不会。

## java.util.Objects
`java.util.Objects`工具类，我觉得好用的几个方法
```java
    public static boolean equals(Object var0, Object var1) {
        return var0 == var1 || var0 != null && var0.equals(var1);
    }
    public static int hashCode(Object var0) {
        return var0 != null ? var0.hashCode() : 0;
    }
    public static <T> T requireNonNull(T var0) {
        if (var0 == null) {
            throw new NullPointerException();
        } else {
            return var0;
        }
    }

    public static <T> T requireNonNull(T var0, String var1) {
        if (var0 == null) {
            throw new NullPointerException(var1);
        } else {
            return var0;
        }
    }        
```
除此之外还应该从`Objects`学习到编写工具类的正确的规范，
* 定义为final class
* 只定义一个无参的构造函数且抛出断言错误，防止被反射调用
* 工具方法都是静态方法
* 静态方法中只抛出unchecked异常





## java.lang.System
这个最早应该是在Hello World程序中见到的，推荐它的一个方法
```java
    /**
     * Returns the same hash code for the given object as
     * would be returned by the default method hashCode(),
     * whether or not the given object's class overrides
     * hashCode().
     * The hash code for the null reference is zero.
     *
     * @param x object for which the hashCode is to be calculated
     * @return  the hashCode
     * @since   JDK1.1
     */
    public static native int identityHashCode(Object x);
```
注释写得很明白了，不管一个对象实例的class有没有覆盖Object的hashCode方法，都能使用这个方法获得hash值。
## 获取泛型类的类型参数
我们可以从以下代码获得提示，代码来自`HashMap`，
```java
    /**
     * Returns x's Class if it is of the form "class C implements
     * Comparable<C>", else null.
     */
    static Class<?> comparableClassFor(Object x) {
        if (x instanceof Comparable) {
            Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
            if ((c = x.getClass()) == String.class) // bypass checks
                return c;
            if ((ts = c.getGenericInterfaces()) != null) {
                for (int i = 0; i < ts.length; ++i) {
                    if (((t = ts[i]) instanceof ParameterizedType) &&
                        ((p = (ParameterizedType)t).getRawType() ==
                         Comparable.class) &&
                        (as = p.getActualTypeArguments()) != null &&
                        as.length == 1 && as[0] == c) // type arg is c
                        return c;
                }
            }
        }
        return null;
    }
```
这里的逻辑是获得类`C`，然后获取它实现的接口`Comparable<C>`，然后从这个`Comparable<C>`中获得类型参数`C`，然后比较这两个类型是否相等。虽然我们一直听说Java的泛型是类型擦除式，但是在这里我们是可以获得泛型的参数类型的。照例用一段demo测试一下，
```java
public class ParameterApp {
    public static void main(String[] args) {
        StringList list = new StringList();
        Class<?> clazz = getTypeArgument(list);
        System.out.println(clazz.getName());
    }

    static Class<?> getTypeArgument(Object x) {
        if (x instanceof Collection) {
            Class<?> c = x.getClass();
            Type[] ts, as; Type t; ParameterizedType p;
            if ((ts = c.getGenericInterfaces()) != null) {
                for (int i = 0; i < ts.length; ++i) {
                    if (((t = ts[i]) instanceof ParameterizedType) &&
                            ((as  = ((ParameterizedType)t).getActualTypeArguments()) != null)
                             &&
                            as.length == 1) // type arg is c
                        return (Class<?>) as[0];
                }
            }
        }
        return null;
    }

    static class StringList extends AbstractList<String> implements List<String> {

        @Override
        public String get(int i) {
            return null;
        }

        @Override
        public int size() {
            return 0;
        }
    }
}
```
## sun.reflect.Reflection
这个工具类是和反射相关的，让大家知道有这么一个方法
```java
    @CallerSensitive
    public static native Class<?> getCallerClass();
```
我第一次见到这个方法是在`java.sql.DriverManager`中的`getConnection`方法中见到的
```java
    @CallerSensitive
    public static Connection getConnection(String url,
        String user, String password) throws SQLException {
        java.util.Properties info = new java.util.Properties();

        if (user != null) {
            info.put("user", user);
        }
        if (password != null) {
            info.put("password", password);
        }

        return (getConnection(url, info, Reflection.getCallerClass()));
    }
```
`Reflection.getCallerClass()`是一个`native`方法，返回的是`Class<?>`类型，在`DriverManager`中使用它的目的是为了获得相应的`ClassLoader`，上面的代码是在Java 8中见到的。其中在Java 7中为获得`ClassLoader`，`DriverManager`就直接提供了`native`的方法
```java
/* Returns the caller's class loader, or null if none */
private static native ClassLoader getCallerClassLoader();
```

我们用一段代码尝试调用这个方法
```java
public class CalleeApp {

    public void call() {
        Class<?> clazz = Reflection.getCallerClass();
        System.out.println("Hello " + clazz);
    }
}
```
```java
public class CallerApp {

    public static void main(String[] args) {
        CalleeApp app = new CalleeApp();
        Caller1 c1 = new Caller1();
        c1.run(app);
    }

    static class Caller1 {
        void run(CalleeApp calleeApp) {
            if (calleeApp == null) {
                throw new IllegalArgumentException("callee can not be null");
            }
            calleeApp.call();
        }
    }

}
```
执行main方法会抛出异常
```java
Exception in thread "main" java.lang.InternalError: CallerSensitive annotation expected at frame 1
```
这个错误信息说的是我们缺少在函数调用栈开始位置添加`CallerSensitive`注解，观察`DriverManager`的`getConnection`方法确实是有这么个注解的。
那如果给`CalleeApp`的`call`加上注解，那结果又会怎样呢？

## Object.wait(long timeout, int nanos)
这个方法是来卖萌，它的本义在注释是这样子写的，
```java
    /*
     * <p>
     * This method is similar to the {@code wait} method of one
     * argument, but it allows finer control over the amount of time to
     * wait for a notification before giving up. The amount of real time,
     * measured in nanoseconds, is given by:
     * <blockquote>
     * <pre>
     * 1000000*timeout+nanos</pre></blockquote>
     * <p>
     */
```
意思是提供精细化的时间衡量，`nano`可是纳秒单位啊！！！
而它的实现却是这样的，
```java
    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos > 0) {
            timeout++;
        }

        wait(timeout);
    }
```
除了对传入参数的数值范围校验外，对`nano`的使用紧紧是判断这个变量是否大于0，是则给`timeout`加1，这只是增加了1毫秒的时间，并没有体现出了精细化的地方。

## java.util.Base64

在此之前使用Base64工具可能需要引入apache commons等jar包，Java 8开始集成到JDK中了。
```
String message = "H for Hope!";
byte[] encoded = Base64.getEncoder().encode(message.getBytes());
System.out.println(new String(encoded));
byte decoded[] = Base64.getDecoder().decode(encoded);
System.out.println(new String(decoded));
```
`Base64`是单例模式，`getEncoder`返回的是内部静态类`Encoder`的静态成员`RFC4648`，`getDecoder`返回的是内部静态类`Decoder`的静态成员`RFC4648`；decode和encode当然是对应起来的，除此之外应该看到这样的写法值得学习，get方法中没有synchronized也没有去执行new等操作，所有需要的单例都是在类初始化的时候构造好的了。


## 附
* [you-dont-need-serial](https://wdd.js.org/you-dont-need-serial.html)
* [Reflection.getCallerClass()使用的问题](https://segmentfault.com/q/1010000014119325)