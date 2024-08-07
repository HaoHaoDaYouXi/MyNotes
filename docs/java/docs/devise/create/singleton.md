# 单例（Singleton）

## 问题
确保一个类只有一个实例，并提供该实例的全局访问点。

## 效果
使用私有的构造函数、静态变量各一个，及一个公有的静态函数来实现。

私有构造函数保证了不能通过构造函数来创建对象实例，只能通过公有静态函数返回唯一的私有静态变量。

## 解决方案

### 懒汉式 线程不安全
```java
public class Singleton {

    private static Singleton singleton;

    private Singleton() {}

    public static Singleton getInstance() {
        if (singleton == null) {
            singleton = new Singleton();
        }
        return singleton;
    }
}
```
私有静态变量`singleton`被延迟实例化，这样如果没有用到该类，那么就不会实例化，从而节约资源。

这个实现在多线程环境下是不安全的，因为多个线程能够同时执行`if (singleton == null)`判断，会导致实例化多次`singleton`。

### 懒汉式 线程安全
为了线程安全，需要对`getInstance()`方法加锁，保证在一个时间点只能有一个线程能够进入该方法，避免了实例化多次。

因为加锁了，所以该方法会存在性能问题，不建议使用。
```java
public static synchronized Singleton getInstance() {
    if (singleton == null) {
        singleton = new Singleton();
    }
    return singleton;
}
```

### 饿汉式 线程安全
线程不安全问题主要是被实例化多次，采取直接实例化的方式不会产生线程不安全问题，直接实例化的方式也丢失了延迟实例化带来的节约资源的好处。
```java
private static Singleton singleton = new Singleton();
```

### 双重校验锁 线程安全
`singleton`只需要被实例化一次，之后就可以直接使用了。

加锁操作只需要对实例化那部分的代码进行，只有没有被实例化时，才需要进行加锁。

双重校验锁先判断是否已经被实例化，如果没有被实例化，那么才对实例化语句进行加锁。
```java
public class Singleton {

    private volatile static Singleton singleton;

    private Singleton() {
    }

    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
第一个`if`语句用来避免`singleton`已经被实例化之后的加锁操作，
第二个`if`语句进行了加锁，所以只能有一个线程进入，就不会出现`singleton == null`时两个线程同时进行实例化操作。

`singleton`采用`volatile`关键字修饰也是很重要的

`singleton = new Singleton();`这段代码是分为三步执行的：
- 为`singleton`分配内存空间
- 初始化`singleton`
- 将`singleton`指向分配的内存地址

由于`JVM`具有指令重排的特性，执行顺序有可能变成`1`>`3`>`2`。
指令重排在单线程环境下不会出现问题，但是在多线程环境下会导致一个线程获得还没有初始化的实例。

例如，线程`T1`执行了`1`和`3`，此时`T2`调用`getSingleton()`后发现`singleton`不为空，
因此返回`singleton`，但此时`singleton`还未被初始化。

使用`volatile`可以禁止`JVM`的指令重排，保证在多线程环境下也能正常运行。

### 静态内部类实现
```java
public class Singleton {

    private Singleton() {}

    private static class SingletonHolder {
        private static final Singleton SINGLETON = new Singleton();
    }

    public static Singleton getSingleton() {
        return SingletonHolder.SINGLETON;
    }
}
```
当`Singleton`类被加载时，静态内部类`SingletonHolder`没有被加载进内存。
只有当调用`getSingleton()`方法从而触发`SingletonHolder.SINGLETON`时`SingletonHolder`才会被加载，
此时初始化`SINGLETON`实例，并且`JVM`能确保`SINGLETON`只被实例化一次。

这种方式不仅具有延迟初始化的好处，而且由`JVM`提供了对线程安全的支持。

### 枚举实现
```java
public enum Singleton {
    SINGLETON;

    private String name;
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public static void main(String[] args) {
        // 测试
        Singleton firstSingleton = Singleton.SINGLETON;
        firstSingleton.setName("firstName");
        System.out.println(firstSingleton.getName());
        Singleton secondSingleton = Singleton.SINGLETON;
        secondSingleton.setName("secondName");
        System.out.println(firstSingleton.getName());
        System.out.println(secondSingleton.getName());

        // 反射获取实例测试
        try {
            
            Singleton[] enumConstants = Singleton.class.getEnumConstants();
            for (Singleton enumConstant : enumConstants) {
                System.out.println(enumConstant.getName());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：
```log
firstName
secondName
secondName
secondName
```

该实现可以防止反射攻击。

在其它实现中，通过`setAccessible()`方法可以将私有构造函数的访问级别设置为`public`，
然后调用构造函数从而实例化对象，如果要防止这种攻击，需要在构造函数中添加防止多次实例化的代码。

该实现是由`JVM`保证只会实例化一次，因此不会出现上述的反射攻击。

该实现在多次序列化和序列化之后，不会得到多个实例。
而其它实现需要使用`transient`修饰所有字段，并且实现序列化和反序列化的方法。

----
