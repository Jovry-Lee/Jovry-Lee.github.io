---
title: DesignPattern-Adapter(适配器模式)
date: 2020-09-29 11:17:34
tags: ["DesignPattern","Adapter"]
categories: ["DesignPattern"]
---

1 概述

适配器其实很好理解，生活中也其实充满了适配器，比如：壁式插座是三角孔，而标准的交流电插头是两头的，若需要将两头插头插入三角孔内，则可能需要一个交流电适配器．

<!--more-->

该交流电的适配器的作用：位于插头和插座之间，将插头转换为三角插头，将交流电经过一定的转换以匹配插座．



2 Adapter（适配器模式）

定义：适配器模式将一个类的接口，转换成客户期望的另一个接口．适配器让原本接口不兼容的类可以合作无间．



作用：

- ①、通过创建适配器进行接口转换，让不兼容的接口变成兼容，让客户从实现的接口解耦。
- ②、若后续有改变接口，适配器可以将改变部分封装起来，无需客户进行修改。



适配器的分类：
- 对象适配器——“组合”方式适配
- 类适配器——“继承”方式适配



| 优缺点     | 优点                                                         | 缺点                          |
| ---------- | ------------------------------------------------------------ | ----------------------------- |
| 对象适配器 | ①、采用组合的方式，步进可以适配某个类，也可以适配该类的任何子类。<br />②、只需要写很少的代码，将工作委托给适配者，更加有弹性。 | ①、需重新实现整个被适配者。   |
| 类适配器   | ①、采用继承的方式，无需重新实现整个被适配者，且可以覆盖被适配者的行为。<br/>②、只需要一个类适配器，不需要一个适配器和一个被适配者。 | ①、只能采用某个特定被适配类。 |



2.1 对象适配器

对象适配器是通过＂组合＂方式适配．其类图如下：

![Adapter_UML](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/Adapter_UML.png)



对象适配器优点：

- ①、采用组合的方式，步进可以适配某个类，也可以适配该类的任何子类。
- ②、只需要写很少的代码，将工作委托给适配者，更加有弹性。



对象适配器缺点：

- ①、需重新实现整个被适配者。



2.2 类适配器

类适配器采用＂继承＂的方式适配．（*类适配器需要多重继承才能实现，而对于Java, PHP是不支持多重继承的*）



其类图如下：

![AdapterClassUml](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/AdapterClassUml.png)



类适配器的优点：

- ①、采用继承的方式，无需重新实现整个被适配者，且可以覆盖被适配者的行为。
- ②、只需要一个类适配器，不需要一个适配器和一个被适配者。



类是配器的缺点：

- ①、只能采用某个特定被适配类。



3 适配器模式的应用

假设有一个带有`fly()`和gobble()方法的Turkey类。还有一个带有fly()和quack()方法的Duck类。让我们假设你是Duck对象的简称，你想在他们的位置使用Turkey对象。Turkey有一些类似的功能，但实现了不同的接口，所以我们不能直接使用它们。所以我们将使用适配器模式。在这里，我们的客户将是Duck，而adaptee将是Turkey。

（*由于PHP并不支持多重继承，因此此处只分析对象适配器的情况*）

其类图如下：

![AdapterObjectExample](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/AdapterObjectExample.png)



```php
// 鸭子接口,具备呱呱叫和飞行的能力.
interface Duck
{
    public function quack(); // 鸭子叫.
    public function fly();  // 飞行能力.
}

// 火鸡接口.
interface Turkey
{
    public function gobble(); // 火鸡咯咯叫.
    public function fly(); // 火鸡会飞,但飞不远.
}

// 绿头鸭是鸭子的实现.
class MallardDuck implements Duck
{
    public function quack()
    {
        echo "Quack\n";
    }

    public function fly()
    {
        echo "I'm flying\n";
    }
}

// 野火鸡是火鸡的实现.
class WildTurkey implements Turkey
{

    public function gobble()
    {
        echo "Gobble gobble\n";
    }

    public function fly()
    {
        echo "I'm flying a short distance\n"; // 火鸡只能飞很短的距离．
    }
}

// 鸭子适配器(让火鸡来充当鸭子),首先需要实现想要转换成的类型接口,也就是客户所期望看到的接口.
class TurkeyAdapter implements Duck
{
    /**@var Turkey $turkey 传入的火鸡对象*/
    private $turkey;

    // 接着需要取得适配的对象引用.
    public function __construct(Turkey $turkey)
    {
        $this->turkey = $turkey;
    }

    // 实现接口中的所有方法,试下转换.
    public function quack()
    {
        $this->turkey->gobble();　// 在鸭子的相应方法里面，调用传入的火鸡对象的接口．
    }

    // 虽然两个接口都具备了fly方法,火鸡的飞行距离短,不像鸭子可以长途飞行,要让鸭子的飞行和火鸡的飞行对应,必须连续调用火鸡的fly来完成.
    public function fly()
    {
        for ($i = 0; $i < 5; $i++) {
            $this->turkey->fly();
        }
    }

}

// 测试.
$duck = new MallardDuck();　// 创建一只鸭子.
$turkey = new WildTurkey();　//  创建一只火鸡.

// 将火鸡包装进一个火鸡适配器,使它看起来象一只鸭子.
$turkeyAdapter = new TurkeyAdapter($turkey);
echo "The Turkey says...\n";
$turkey->gobble();　// Gobble gobble
$turkey->fly(); // I'm flying a short distance

echo "\nThe Duck says...\n";
$duck->quack(); // Quack
$duck->fly(); // I'm flying

echo "\nThe TurkeyAdapter says...\n";
$turkeyAdapter->quack(); // Gobble gobble
$turkeyAdapter->fly(); // I'm flying a short distance\nI'm flying a short distance\nI'm flying a short distance\nI'm flying a short distance\nI'm flying a short distance
```



------

参考资料
Head First Design Pattern
适配器模式（https://www.geeksforgeeks.org/adapter-pattern/）