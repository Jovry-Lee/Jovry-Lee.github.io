---
title: DesignPattern-Singleton
date: 2020-09-21 16:34:15
tags: ["DesignPattern","Singleton"]
categories: ["DesignPattern","Singleton"]
---



### 1 概述

有一些对象其实我们只需要一个，比如说：线程池（threadpool）、缓存（cache）、对话框、处理器偏好设置和注册表的对象、日志对象、充当打印机、显卡等设备的驱动程序的对象。如果制造多个实例可能会出现许多问题，例如:程序行为异常、资源使用过量，或者不一致的结果．这就引入了单例模式．

<!--more-->

实际上，在不同通过单例模式实例化一个对象外，还可以有很多方式可以实现，例如：可以使用全局变量实现仅实例化一个对象（即在程序一开始就创建号该对象）．但是若该对象非常消耗资源，而程序有一直没有用到它，就形成了资源的浪费．而利用单例模式则可以在需要的时候才创建对象．



### 2 Singleton（单例模式）

`单例模式的定义`：确保一个类只有一个实例，并提供了一个全局访问点．

`单例模式的实现原理`：

- 利用一个静态变量来记录Singleton的唯一实例．
- 将构造器声私有化，只有Singleton类内才可以调用构造器．
- 提供一个静态的方法（eg：`getInstance()`）实例化对象，并返回这个实例．



其类图如下：

![Singleton_类图](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/Singleton_类图.png)



### 3 单例模式实现方法

#### 3.1 延迟实例化方法（饱汉模式）

`延迟实例化方法`是最经典的实现方式，对静态变量的实例化赋值放在静态方法中实现．



PHP实现示例：

```php
class Singleton
{
    private static $instance; // 不进行赋值．
    private function __construct(){}　//　私有化构造器．

    public static function getInstance()　
    {
        if (static::$instance == null) {
            static::$instance = new self();　// 需要使用该实例的时候，再进行赋值．
        }
        return static::$instance;
    }
}
// 调用方法．
$singleton = Singleton::getInstance();
```



`问题`：该方式在单进程模式下，可以有效确保一个类只有一个实例，但是在多进程并发的情况下，可能会导致多个实例产生．

解决方法：

- 使用同步的方法．但同步的方法可能会造成程序执行效率下降．
  - 通过增加synchronized关键字到getInstance方法中，迫使每个线程在进入到这个放大之前，要先等待别的线程离开该方法．
- 不采用延迟实例化方法（饿汉模式）．
- 使用＂双重检测加锁＂.（使用 volatile实现）



#### 3.2 饿汉模式

顾名思义，即在静态初始化阶段中创建该示例，保证代码在调用getInstance时已经有实例了．



`PHP实现示例`：

```php
class Singleton
{
    private static $instance = new self();
    private function __construct(){}

    public static function getInstance()
    {
        return static::$instance;
    }
}

$singleton = Singleton::getInstance();
```



------

### 参考资料：

Head First设计模式 第五章（P170）



































































