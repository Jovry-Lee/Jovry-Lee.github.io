---
title: PHP-Laravel安装及使用
date: 2020-10-05 16:43:13
tags: ["PHP","Laravel"]
categories: ["PHP","Laravel"]
---

Laravel是一套简洁、优雅的PHP Web开发框架(PHP Web Framework)。

<!--more-->

#### 1 安装

##### 1.1 服务器要求

确保服务器满足版本要求且已安装以下扩展：`BCMath`，`Ctype`,`Fileinfo`,`JSON`,`Mbstring`,`OpenSSL`,`PDO`,`Tokenizer`,`XML`

##### 1.2 安装Laravel

- 使用composer安装Laravel安装器

```
composer global require laravel/installer
```

- 修改系统环境变量

```txt ~/.bashrc
export LARAVEL_HOME=/root/.composer/vendor/bin
export PATH=$LARAVEL_HOME:$PATH
```

- 执行以下命令使其生效

```
source ~/.bashrc
```

##### 1.3  创建Laravel项目

###### 1.3.1 使用`laravel new`命令

示例：通过以下命令将创建一个名为blog的目录，并安装号Laravel所有依赖项。

```
laravel new blog
```

###### 1.3.2 通过Composer创建项目

通过`create-project`命令来安装Laravel：

```
composer create-project --prefer-dist laravel/laravel blog <版本>
```

示例：由于我本机php版本为7.1.3，查看支持的Laravel版本，对应的应该为5.6

```
composer create-project --prefer-dist laravel/laravel blog 5.6.*
```

##### 1.4 启动php内置服务器

通过以下命令，将在`http://localhost:8000`上启动开发服务器。

```
php artisan serve
```



#### 2 配置

##### 2.1 配置storage和bootstrap/cache目录权限

安装完Laravel后，需要配置storage及bootstrap/cache目录权限，使其允许Web服务器写入，否则，Laravel程序将无法使用。

##### 2.2 配置应用秘钥

 命令：`php artisan key:generate`

注：若Laravel是通过Composer或Laravel安装器安装的，则无需再次进行设置。

若未设置好应用秘钥，则用户会话和其他加密数据是不安全的。

##### 2.3 环境配置

Laravel提供了一个.env文件用来管理不同环境的配置，该配置不应被提交到版本控制系统中。

本地开发时，可以创建一个测试env文件，例如：`.env.tesing`文件，当使用PHPUnit测试或者以`--env=testing`为选项执行Artisan命令，该文件将覆盖`.env`文件中的值。

##### 2.4 配置缓存

Laravel提供了一个缓存配置命令，将所有的配置文件缓存到单个文件中，后续框架会快速加载该文件，命名如下：

```
php artisan config:cache
```

注：该命令应该作为生产环境部署中，但不应在本地开发环境执行，一旦配置被缓存，`.env`文件将不再被加载，所有对`env`函数的调用将返回null



#### 3 依赖注入（服务容器）

Laravel提供了一个服务容器，用于管理类依赖以及实现依赖注入。

##### 3.1 常用注入方式

###### 3.1.1 简单注入（简单绑定）

Laravel提供了`bind`方法，用于注册绑定，其第一个参数为要绑定的`类/接口名`，第二个参数是一个返回类实例的`Closure`。

示例：

```
$this->app->bind('HelpSpot\API', function ($app) {
    return new \HelpSpot\API($app->make('HttpClient'));
});
```

###### 3.1.2 注入单例

Laravel提供了一个`singleton`方法，将类或接口绑定到只解析一次的容器中。一旦被绑定，相同的对象实例会在随后的调用中返回。

示例：

```
$this->app->singleton('HelpSpot\API', function ($app) {
    return new \HelpSpot\API($app->make('HttpClient'));
});
```

###### 3.1.3 注入实例

Laravel提供了一个`instance`方法将现有对象实例绑定到容器。给定的实例会始终在随后的调用中返回到容器。

示例：

```
$api = new \HelpSpot\API(new HttpClient);
$this->app->instance('HelpSpot\API', $api);
```

###### 3.1.4 注入接口到实现

Laravel支持绑定接口到给定的实现。

示例：

```
$this->app->bind(
    'App\Contracts\EventPusher',
    'App\Services\RedisEventPusher'
);
```

##### 3.2 解析

###### 3.2.1 解析类的实例

Laravel提供了一个make方法用于从容器中解析出类或接口的名称。

- 从app变量中解析：`$this->app->make('HelpSpot\API')`
- 通过全局辅助函数解析：`resolve(HelpSpot\API)`
- 通过关联数组作为makeWith方法的参数注入：`$this->app->makeWith('HelpSpot\API', ['id' => 1])`

###### 3.2.2 自动注入

通过构造函数进行注入

##### 3.3 容器事件

Laravel服务容器每次解析对象会触发一个事件 ，可以通过使用`resoving`方法监听这个事件。

```
$this->app->resolving(function ($object, $app) {
    // 当容器解析任何类型的对象时调用...
});

$this->app->resolving(\HelpSpot\API::class, function ($api, $app) {
    // 当容器解析类型为 "HelpSpot\API" 的对象时调用...
});
```



#### 4 Service Provider（服务提供者）

##### 4.1 Service Provider的编写方法

所有的服务提供者都会继承`Illuminate\Support\ServiceProvider`类。大多数提供者包含

- register（注册方法）：需将服务绑定到服务容器中（注：不要在register方法中注册监听器、路由或其他功能，否则会议外地使用到尚未加载的服务提供者提供的服务）。
- boot（引导方法）：该方法在所有服务提供者被注册后才调用。

生成提供者命令：

```
php artisan make:provider <服务提供者名称>
```



服务提供者提供了两个简单绑定的特性，当服务提供器被框架加载时，将自动检查这些属性并注册相应的绑定：

- bindings
- singletons

##### 4.2 Service Provider的注册方法

将Service Provider配置到配置文件`config/app.php`中的`providers`数组中即可进行注册。

##### 4.3 延迟加载Service Provider

延迟加载Service Provider需要实现`\Illuminate\Contracts\Support\DeferrableProvider` 接口并置一个 `provides` 方法，然后在`provides`方法中返回该提供者注册的服务容器绑定。



#### 5 Facades

Laravel Facades 实际是服务容器中底层类的 “静态代理” ，相对于传统静态方法，在使用时能够提供更加灵活、更加易于测试、更加优雅的语法。所有的 Laravel Facades 都定义在 `Illuminate\Support\Facades` 命名空间下。











































































