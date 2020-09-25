---
title: DesignPattern-Factory
tags: ["DesignPattern","Factory"]
categories: ["DesignPattern"]
date: 2020-09-21 17:54:27
---

### 1 概述

当使用＂`new`＂实例化一个具体类，将会导致代码更加脆弱，缺乏弹性．例如：当有很多相关的具体类时（如下代码所示），究竟要实例化哪个类，要在运行时由一些条件来决定．一旦有变化扩者扩展，就需要重新对这段代码进行检查和修改．可能造成部分系统维护和更新的困扰，也容易出错．

<!--more-->

```java
Duck duck;

if (picnic) {
	duck = new MallardDuck();
} else if (hunting) {
	duck = new DecoyDuck();
} else if(inBathTub) {
	duck = new RubberDuck();
}
```



那么如何将实例化具体类的代码从应用总抽离，或者封装起来，使他们不会干扰应用的其他部分呢？

- 识别变化的方面．
- 封装创建对象的代码（该对象即为＂Factory，工厂＂）．



### 2 Factory（工厂模式）

根据应用场景和复杂度的不同，工厂模式可以分为以下三种：

- 简单工厂模式；
- 工厂方法模式；
- 抽象工厂模式；



#### 2.1 简单工厂模式

简单工厂模式，准确点来说，并不算是一种设计模式，反而比较像一种编程习惯．

实现方法：<u>新建一个工厂类，在工厂类内定义一个实例化各种对象的方法，将识别出来变化的部分移植到该方法中．</u>



简单工厂模式看似只是把问题搬到一个对象中了，并没有进行实际的解决，`这样做的好处是什么`？

- 将变化部分的代码封装起来，后续有改变时，只需要改动这一个类，避免对其他逻辑的影响．
- 可以有多个调用方



PHP示例：

```php
// 具体商品，白猫
class WhiteCat{
    function voice() {
        echo "白猫喵喵..";
    }
}

// 具体商品，黑猫
class BlackCat{
    function voice() {
        echo "黑猫喵喵..";
    }
}

// 工厂类，用于生产商品
class factory {
    // 可以通过一个静态方法，通过参数来区分要实例化的类．
    public static function create($color) {
        switch ($color) {
            case 'white':
                return new WhiteCat();
                break;
            case 'black':
                return new BlackCat();
                break;
        }
    }
}

// 模拟客户端
class Client{
    public static function main(){
        $wCat = factory::create('white');
        $wCat->voice();
        $bCat = factory::create('black');
        $bCat->voice();
    }
}

Client::main();
```



#### 2.2 工厂方法模式（Factory Method Pattern）

`工厂方法模式定义了一个创建对象的接口，但由子类决定要实例化的类是哪一个．工厂方法让类的实例化推迟到子类．`



实现方法：<u>通过在抽象类（Creator）中定义抽象的工厂方法（factoryMethod），让其子类（ConcreteCreator）实现此方法以创建实例．（抽象类中通常会包含依赖子类实现的工厂方法，但无须知道是在创建哪种具体的对象．）</u>

（注：Creator类并不是必须为抽象的，可实现其工厂方法作为默认方法，子类若有变化，则重写该方法）



其类图如下：

![FactoryMethodPattern](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/FactoryMethodPattern.png)



PHP示例：

```php
// 商品基类
abstract class Cat{
    abstract protected function voice();
}

// 具体商品WhiteCat
class WhiteCat extends Cat{
    function voice() {
        echo "白猫喵喵..";
    }
}

// 具体商品BlackCat
class BlackCat extends Cat{
    function voice() {
        echo "黑猫喵喵..";
    }
}

// 工厂接口，用于实例化
interface factory{
    // static主要是为了不实例子工厂类，直接调用create方法
    public static function create();
}

// 白猫工厂，用于实例化白猫
class WhiteCatFactory implements factory{
    public static function create() {
        return new WhiteCat();
    }
}

// 黑猫工厂，用于实例化黑猫
class BlackCatFactory implements factory{
    public static function create() {
        return new BlackCat();
    }
}

class Client{
    public static function main(){
        $wCat = WhiteCatFactory::create();
        $wCat->voice();
        $bCat = BlackCatFactory::create();
        $bCat->voice();
    }
}

Client::main();
```





`简单工厂和工厂方法之间的区别？`

- 简单工厂将所有的逻辑，在一个地方处理完；它不具有工厂方法的弹性，因为简单工厂不能变更正在创建的对象．
- 工厂方法是创建一个框架，让子类决定如何实现．



`工厂方法的优点：`

- 将创建对象的代码集中在一个对象或者方法中，可以避免代码的重复，并且更方便以后的维护．
- 实例化对象时，只会依赖于接口，而不是具体类．使代码更具有弹性，方便未来扩展．



#### 2.3 抽象工厂模式

`抽象工厂模式提供一个接口，用于创建相关或依赖对象的家族，而不需要明确指定具体类．`

`实现方法`：

- 抽象工厂定义了一个接口．所有的具体工厂都必须实现此接口，这个接口包含一组方法用来生产产品．
- 所有具体工厂实现不同的产品家族．要创建一个产品，客户只要使用其中一个工厂而完全不需要实例化任何产品对象．



其类图如下：

![AbstractFactory](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/AbstractFactory.png)



PHP示例：

```php
// 商品基类，猫
abstract class Cat{
    abstract protected function voice();
}

// 具体商品WhiteCat
class WhiteCat extends Cat{
    function voice() {
        echo "白猫喵喵..";
    }
}

// 具体商品BlackCat
class BlackCat extends Cat{
    function voice() {
        echo "黑猫喵喵..";
    }
}

// 商品基类，狗
abstract class Dog{
    abstract protected function voice();
}

// 具体商品WhiteDog
class WhiteDog extends Dog{
    function voice() {
        echo "白狗汪汪..";
    }
}

// 具体商品BlackDog
class BlackDog extends Dog{
    function voice() {
        echo "黑狗汪汪..";
    }
}

// 工厂接口，用于实例化,提供一组商品．
interface factory{
    public function createCat();
    public function createDog();
}

// 白色动物工厂，用于实例化白猫，白狗
class WhiteAnimalFactory implements factory{
    public function createCat() {
        return new WhiteCat();
    }
    public function createDog() {
        return new WhiteDog();
    }
}

// 黑色动物工厂，用于实例化黑猫，黑狗
class BlackAnimalFactory implements factory{
    public function createCat() {
        return new BlackCat();
    }
    public function createDog() {
        return new BlackDog();
    }
}

class Client{
    public static function main() {  
        self::run(new WhiteAnimalFactory());  
        self::run(new BlackAnimalFactory());  
    }  

    public static function run(factory $AnimalFactory){  
        $cat = $AnimalFactory->createCat();  
        $cat->Voice();  

        $dog = $AnimalFactory->createDog();  
        $dog->Voice();  
    }
}

Client::main();
```



`抽象工厂模式的优点`：

- 将一群相关的产品集合起来．



`抽象工厂模式的缺点`：

- 扩展相关接口较难，例如新增一个接口．这就意味着需要深入改变每个子类的接口．



### 3 工厂方法与抽象工厂的比较

**关联点**：

- `抽象工厂的方法经常以工厂方法的方式实现`．抽象工厂的任务是定义一个负责创建一组产品的接口．这个接口内的每个方法都则创建一个具体的产品，同时利用实现抽象工厂的子类类提供这些具体的做法．



**相同点**：

- 工厂方法和抽象方法都能将对象的创建封装起来，使应用程序解偶，降低其对特定实现的依赖．



**不同点**：

| 功能                               | 工厂方法               | 抽象工厂                                                     |
| ---------------------------------- | ---------------------- | ------------------------------------------------------------ |
| 创建对象的方法                     | 继承                   | 对象的组合                                                   |
| 解偶方式（将客户从具体类型中解偶） | 通过子类来创建具体对象 | 提供一个用来创建产品家族的抽象类型，这个类型的子类定义了产品被生产的方法．要使用这个工厂，必须先实例化它，然后将它传入一些针对抽象类型所写的代码中． |



### 4 总结

- 所有的工厂都是用来封装对象的创建．通过减少应用程序和具体类之间的依赖促进松耦合．
- 简单工厂，虽然不是真正的设计模式，但它是一种简单的可以将客户程序从具体类中解偶的方法．
- 工厂方法通过使用继承，允许类将实例化延迟到子类，由子类实现工厂方法来创建对象．
- 抽象工厂使用对象组合，对象的创建被是现在工厂接口所暴露出来的方法中．



------

### 参考资料：

Head First设计模式 第五章（P109）

































