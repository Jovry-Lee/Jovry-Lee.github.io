---
title: DesignPattern-Command(命令模式)
date: 2020-09-25 11:17:34
tags: ["DesignPattern","Command"]
categories: ["DesignPattern"]
---



### 1 概述

当我们使用遥控器时，每个按钮对应了不同的功能，对于使用者来说，只需要知道某个按键的功能，并不需要这些功能涉及了哪些硬件设备，具体怎样实现执行相应的功能．

<!--more-->

抽象来看，通过封装方法调用，将＂动作请求者＂从＂动作执行者＂对象中解偶，这就是`Command Pattern（命令模式）`．



### 2 Command（命令模式）

**命令模式**指将＂请求＂封装成对象，以便使用不同的请求，队列或日志来参数化其他对象．命令也支持可撤销的操作．

一个命令对象通过在特定接收者上绑定一组动作来封装一个请求，并只暴露一个execute()方法，方此方法被调用时，接收者就会进行相应的动作．



其类图如下：

![Command_UML](https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/DesignPattern/Command_UML.png)



PHP示例：

```php
// 1. 实现一个命令接口，所有命令对象均需要实现该接口．
interface Command
{
    public function execute();
}

// 2. 当前有一个Light类用于底层控制电灯的开关．
class Light
{
    public function on()
    {
        echo "Light is On\n"; // 实现开灯.
    }

    public function off()
    {
        echo "Light is Off\n";// todo 实现关灯.
    }
}

// 3. 实现一个打开电灯的命令．
class LightOnCommand implements Command
{
    public $light;
	// 构造器被传入了某个电灯，以便让这个命令控制，然后记录在实例变量中，一旦调用execute()．就由这个电灯对象成为接收者，负责接收请求．
    public function __construct(Light $light)
    {
        $this->light = $light;
    }
	// 通过execute()方法调用接收对象的on方法.
    public function execute()
    {
        $this->light->on();
    }
}

// 4.有一个简单的遥控器用于打开电灯．
class SimpleRemoteContorl
{
    // 命令槽, 持有某个命令.这个命令用于控制者某个装置．
    protected $slot;
	// 设置插槽控制命令，若需要更改遥控器的按钮行为，可以多次设置．
    public function setCommand(Command $command)
    {
        $this->slot = $command;
    }
	// 当按下按钮时，这个方法就会被调用．使得当前命令衔接插槽，并调用它的execute()方法．
    public function buttenWasPressed()
    {
        $this->slot->execute();
    }
}

// 5. 测试． 
$simpleRemoteControl = new SimpleRemoteContorl();　// 实例化一个遥控器.
$light = new Light();　// 实例化一个灯.
$lightOnCommand = new LightOnCommand($light);　// 实例化一个开灯的行为.
$simpleRemoteControl->setCommand($lightOnCommand);　// 遥控器设置开灯行为.
$simpleRemoteControl->buttenWasPressed();　// 按下遥控器.
```



#### 2.1 命令模式的优缺点

优点：

- 使代码可扩展，添加新命令无需更改现有的代码。
- 减少命令和调用者的接受者的耦合。



缺点：

- 每增加一个命令都要增加一个类。



### 3 扩展应用

队列或日志请求，并支持可撤销操作 意味着Command的Execute操作可以存储状态，以便在Command本身中反转其效果。Command可能具有添加的unExecute操作，该操作可以反转先前执行调用的影响。它还可以支持日志记录更改，以便在系统崩溃时可以重新应用它们。



#### 3.1 队列请求

假设当前有一个工作队列，在某一端添加命令，然后另一端是线程．线程从队列中取出一个命令，调用它的execute()方法，等待这个调用完成，然后将此命令对象丢弃，再取出下一个命令．



#### 3.2 日志请求

某些应用需要我们将所有的动作都记录在日志中，并能在系统司机后，重新调用这些动作恢复到之前的状态．使用命令模式将能支持这一点．

当我们执行命令的时候，将历史记录存储在磁盘上，一旦系统死机，我们就可以将命令对象重新加载，并程批得依次调用这些对象的execute方法．



### 4 总结

- 命令模式将发出请求的对象和执行请求的对象解耦；
- 在被解耦的两者之间是通过命令对象进行通信的．命令对象封装了接收者和一个或一组动作．
- 调用者通过调用命令对象的execute()发出请求，使得接收者的动作被调用；
- 调用者可以接收命令当作参数，甚至在运行时动态的进行；
- 命令可以支持撤销，做法时实现一个undo()方法来回到execute()被执行前的状态．



------

### 参考资料

First Head设计模式——封装调用：命令模式
[Command Pattern](https://www.geeksforgeeks.org/command-pattern/)



