---
title: DesignPattern-Decorator
date: 2020-09-28 10:08:45
tags: ["DesignPattern","Decorator"]
categories: ["DesignPattern"]
---



### 1 概述

我们知道OOP的＂继承＂功能非常强大，利用继承设计子类的行为，是在编译时静态决定的，而且所有的子类都会继承到相同的行为，达到复用的目的．但实际上使用继承的方式却不总是最有弹性的，可能造成的问题有：类数量爆炸，设计死板，基类加入新功能并不适用与所有的子类．

<!--more-->

利用组合（composition）和委托（delegation）可以在运行时具有继承的效果．通过动态地组合对象，可以写新的代码添加新功能，而无须修改现有代码．既然没有改变现有代码，那么引进bug或产生意外副作用的机会将大大减少．



这里有一个**重要的设计原则**：

> 类应该对扩展开放，对修改关闭

该设计原则的目标是：<u>允许类容易扩展，在不修改现有代码的情况下，可以搭配新的行为．使其具有弹性可以应对改变，可以接受新的功能来应对改变的需求．</u>

装饰者模式就是完全遵守开放-关闭原则的一个好例子．



### 2 装饰者模式

**装饰者模式**动态地将责任附加到对象上．若要扩展功能，装饰者提供了比继承更有弹性的替代方案．



其UML图如下：

![Decorator_UML](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/Decorator_UML.png)



`特征`：

- 装饰者(`Decorator`)和被装饰对象(`ConcreteComponent`)有相同的超类型（`Component`）；
- 可以用一个或多个装饰者包装一个对象；
- 装饰者和被装饰对象有相同的超类型，所以在任何需要原始对象（被包装的）的场合，都可以用装饰过的对象代替它．
- 装饰者可以在所委托被装饰者的行为之前与/或之后，加上自己的行为，以达到特定的目的．
- 对象可以在任何时候被装饰，所以可以在运行时动态地，不限量的用装饰者来装饰对象．



#### 2.1 装饰器模式中继承和组合的关系

从装饰器模式的类图可见．`Decorator`继承了`Component`，这里用到继承．但是这里实际上是让装饰者和被装饰者拥有相同的超类．此处的继承是为了＂类型匹配＂，而不是＂继承行为＂；



### 3 装饰器模式示例

根据以下类图实现一个简单的饮料系统：

![DecoratorExample](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/DecoratorExample.png)



> 在Head First设计模式书的示例中，Beverage抽象类中包含一个getDescription和cost两个方法，其中cost方法是抽象方法；CondimentDecorator抽象类继承于Beverage类，并将getDescription这个非抽象方法重载为抽象方法．
>
> 
>
> 对于PHP编码报出以下错误：PHP Fatal error:  Cannot make non abstract method Beverage::getDescription() abstract in class CondimentDecorator
>
> 即对于PHP语言，不能将一个已声明为非抽象的方法，重新声明为抽象方法．



```php
// Beverage是一个抽象类．
abstract class Beverage
{
    public $description = "Unknown Beverage";
    public function getDescription()
    {
        return $this->description;
    }
    // 子类必须实现此方法．
    public abstract function cost();
}

/* 
// 由于PHP不支持将非抽象方法声明为抽象方法，因此测试时，先将该方法注释掉．
// 装饰器类必须与被装饰类拥有相同的超类.
abstract class CondimentDecorator extends Beverage
{
    // 所有的装饰者都必须重新实现getDescription方法.
    public abstract function getDescription();
}
*/
// 浓缩咖啡．
class Espresso extends Beverage
{
    public function __construct()
    {
        $this->description = 'Espresso';
    }

    public function cost()
    {
        return 1.99;
    }
}

class HouseBlend extends Beverage
{
    public function __construct()
    {
        $this->description = 'House Blend Coffee';
    }

    public function cost()
    {
        return 0.89;
    }
}

// Mocha是一个装饰者，所以让他继承CondimentDecorator
class Mocha extends Beverage
{
    /** @var Beverage $beverage 用于让Mocha引用一个Beverage*/
    public $beverage;

    // 将Beverage实例传给Mocha的实例变量中.
    public function __construct(Beverage $beverage)
    {
        $this->beverage = $beverage;
    }

    // 添加Mocha的描述．
    public function getDescription()
    {
        return $this->beverage->getDescription() . ", Mocha";
    }

    // 添加摩卡的价格．
    public function cost()
    {
        return 0.2 + $this->beverage->cost();
    }
}

// 创建一个被浓缩咖啡．
$beverage = new Espresso();
echo "{$beverage->getDescription()} \${$beverage->cost()}\n";
// 做成一杯摩卡．
$beverage = new Mocha($beverage);
echo "{$beverage->getDescription()} \${$beverage->cost()}\n";

$beverage１ = new HouseBlend();
echo "{$beverage１->getDescription()} \${$beverage１->cost()}\n";

$beverage１ = new Mocha($beverage１);
echo "{$beverage１->getDescription()} \${$beverage１->cost()}\n";
```



### 4 装饰器模式的应用

在JAVA语言中，java.io包里面很多都应用了装饰器模式，例如`FileInputStream-->BufferedInputStream-->LineNumberInputStream`，其中：

- FileInputStream是被装饰的组件，Java I/O程序库提供了几个组件，包括FileInputStream，StringBufferInputStream，ByteArrayInputStream等，这些都提供了最基本的字节读取功能；
- BufferedInputStream是一个具体的装饰者，它加入来嗯中行为：利用缓冲输入来改进性能，用一个readline()方法来增强接口；
- LineNumberInputStream也是一个具体的装饰者，他加上了计算行数的能力．



如下图所示：

![JavaIO-Decorator](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/JavaIO-Decorator.png)



------

### 参考资料

Head First设计模式 装饰对象：装饰者模式（P79）