---
title: PHP7内核-FPM
date: 2020-08-21 18:15:16
tags: ["PHP"]
categories: ["PHP"]
---



### 1 概述
FPM（FastCGI Process Manager）是PHP FastCGI运行模式的一个进程管理器， 其<u>核心功能是进程管理</u>。  
FastCGI是Web服务器（如Nginx，Apache）和处理程序之间的一种通信协议， 类似于Http，是一种应用层通信协议。<u>注：FastCGI只是一种协议</u>。

<!--more-->

PHP处理Http请求过程：`PHP接收请求`、`解析协议`，`处理完成返回请求`。  
在网络应用场景下，PHP实现了FastCGI协议，然后与web服务器配合实现了http的处理，web服务器处理http请求，然后将解析的结果通过FastCGI协议转发给处理程序，处理程序处理完成后将结果返回给web服务器，web服务器再返回给用户，如下图所示：
![fastcgi](https://note.youdao.com/yws/api/personal/file/94A461E9A2D44F08BCA476D311390436?method=download&shareKey=b6ccb2591612fc2c28386720a51330f4)

<u>PHP实现了FastCGI协议的解析，但未具体实现网络处理</u>，一般的处理模型：`多进程`，`多线程`。

- **多进程模型**：主进程只负责管理子进程，而基本的网络事件由各个子进程处理，例如：nginx、fpm。
- **多线程模型**：与多进程类似，只是它是线程粒度，这种模式通常由主线程监听、接收请求，然后交给子线程处理，例如：memcache。有的也用多进程的那种模式——主线程只负责管理子线程，各个子线程负责监听、接收、处理请求，例如：memcache使用udp协议的情况。

*进程拥有独立的地址空间及资源，而线程没有，线程之间共享进程的地址空间及资源，所以在资源管理上多进程模型比较简单，而多线程模型需考虑不同线程之间的资源冲突，及线程安全。*

### 2 基本实现
FPM是一个`多进程模型`，它由`一个master进程`和`多个worker进程`组成。master会创建一个socket，但不会接口处理进程，而是由fork出的worker进程处理接收请求和处理。

- **master进程**：master进程的主要工作是管理worker进程，负责fork或kill掉worker进程。
- **worker进程**：worker进程的主要工作是处理请求，其生命周期为：`accept请求->解析FastCGI->执行相应脚本->关闭请求->等待新的请求`。  
*注：Fpm为阻塞式模型，即一个进程只会同时链接一个请求。（目的是为了简化PHP的资源管理，使得在Fpm模式下不需要考虑并发导致的资源冲突）*

**FPM的实现概括**：创建一个master进程，在master进程中创建并监听socket， 然后fork出多个子进程，这些子进程各自accept请求，有请求达到后开始读取请求数据，读取完成后开始处理然后返回。<u>（子进程启动后阻塞在accept上，直到有请求到达，且子进程同时只能响应一个请求。）</u>

FPM的master进程与worker进程之间不会直接进行通信，master通过`共享内存`获取worker进程的信息（worker当前状态、已处理请求数等），当master进程要kill一个worker进程则通过`发信号的方式`通知worker进程（**master进程管理woker进程通过发信号的方式**）。

<u>FPM可以监听多个端口，每个端口对应一个worker pool，而每个pool下对应多个worker进程。</u>
![worker_pool](https://note.youdao.com/yws/api/personal/file/54B0F0FF2171453D83B2C319E5A110B9?method=download&shareKey=fbf52aed774d116102a91aae2ab8f1bd)

在`php-fpm.conf`（php-fpm.conf路径：`/etc/php/7.1/fpm/php-fpm.conf`）中通过[pool name]声明一个worker pool：  
（php-fpm.conf文件中include了多个pool配置，相关配置在（`/etc/php/7.1/fpm/pool.d/*.conf`））

```
[web1]
listen = 127.0.0.1:9000
...

[web2]
listen = 127.0.0.1:9001
...
```

启动fpm后查看进程：ps -aux|grep fpm
```
root     27155  0.0  0.1 144704  2720 ?        Ss   15:16   0:00 php-fpm: master process (/usr/local/php7/etc/php-fpm.conf)
nobody   27156  0.0  0.1 144676  2416 ?        S    15:16   0:00 php-fpm: pool web1
nobody   27157  0.0  0.1 144676  2416 ?        S    15:16   0:00 php-fpm: pool web1
nobody   27159  0.0  0.1 144680  2376 ?        S    15:16   0:00 php-fpm: pool web2
nobody   27160  0.0  0.1 144680  2376 ?        S    15:16   0:00 php-fpm: pool web2
```

**具体实现**：
`worker pool`通过`fpm_worker_pool_s`这个结构表示，多个`worker pool`组成一个**单链表**:

```c
struct fpm_worker_pool_s {
    struct fpm_worker_pool_s *next; //指向下一个worker pool
    struct fpm_worker_pool_config_s *config; //conf配置:pm、max_children、start_servers...
    int listening_socket; //监听的套接字
    ...

    struct fpm_child_s *children; // 当前pool的worker链表，每一个worker对应一个fpm_child_s结构
    int running_children; //当前pool的worker运行总数
    int idle_spawn_rate;
    int warn_max_children;

    struct fpm_scoreboard_s *scoreboard; //记录worker的运行信息，比如空闲、忙碌worker数
    ...
}
```

#### 2.1 FPM的初始化
Fpm在启动后首先会进行`SAPI的注册操作`，接着会进入PHP生命周期的`module startup`阶段，在这个阶段会调用各个扩展定义的MINT钩子函数，然后进行一系列的初始化操作，最后master，worker进程进入不同的处理环节。

<u>fpm的启动流程：</u>

```c
//sapi/fpm/fpm/fpm_main.c
int main(int argc, char *argv[])
{
    ...
    //注册SAPI:将全局变量sapi_module设置为cgi_sapi_module
    sapi_startup(&cgi_sapi_module);
    ...
    //执行php_module_starup()
    if (cgi_sapi_module.startup(&cgi_sapi_module) == FAILURE) {
        return FPM_EXIT_SOFTWARE;
    }
    ...
    //初始化
    if(0 > fpm_init(...)){
        ...
    }
    ...
    fpm_is_running = 1;

    fcgi_fd = fpm_run(&max_requests);//后面都是worker进程的操作，master进程不会走到下面
    parent = 0;
    ...
}
```
`fpm_init()`主要有以下几个关键操作：

- fpm_conf_init_main():  解析php-fpm.conf配置文件.
解析`php-fpm.conf`配置文件，分配worker pool内存结构并保存到全局变量中：fpm_worker_all_pools，各worker pool配置解析到`fpm_worker_pool_s->config`中，以下为config中的几个常用配置：
```c
struct fpm_worker_pool_config_s {
	char *name; // pool名称，即配置：[pool name]
	char *user; // Fpm的启动用户：配置：user
	char *group; // 配置：group
	char *listen_address; // 监听的地址，配置：listen
	...
	int pm; // 进程模型：static、dynamic、ondemand
	int pm_max_children; // 最大worker进程数
	int pm_start_servers; // 启动时初始化的worker数
	int pm_min_spare_servers; // 最小空闲worker数
	int pm_max_spare_servers; // 最大空闲worker数
	int pm_process_idle_timeout; // worker空闲时间
	int pm_max_requests; // worker处理的最多请求数，超多这个值worker将被kill
    ...
};
```

- fpm_scoreboard_init_main():  分配用于记录worker进行运行信息的共享内存.
分配用于`记录worker进程运行信息的共享内存`。按照worker pool的最大worker进程数分配，每个worker pool分配一个**fpm_scoreboard_s**结构，pool下对应的每个worker进程分配一个**fpm_scoreboard_proc_s**结构，各结构的对应关系如下图。
![worker_pool_struct](https://note.youdao.com/yws/api/personal/file/E8A679F76DF54280A4A4760C55D94B7A?method=download&shareKey=07fc4d0f2f25c4df01ae4109f6b2d738)

- fpm_signals_init_mian():  mataer进行创建管道及注册信号管理worker进程.
```c
static int sp[2];

int fpm_signals_init_main()
{
    struct sigaction act;

    //创建一个全双工管道，该管道不是用于master与worker进程通信的，只在master进程中使用。
    if (0 > socketpair(AF_UNIX, SOCK_STREAM, 0, sp)) {
        return -1;
    }
    //注册信号处理handler
    act.sa_handler = sig_handler;
    sigfillset(&act.sa_mask);
    if (0 > sigaction(SIGTERM,  &act, 0) ||
        0 > sigaction(SIGINT,   &act, 0) ||
        0 > sigaction(SIGUSR1,  &act, 0) ||
        0 > sigaction(SIGUSR2,  &act, 0) ||
        0 > sigaction(SIGCHLD,  &act, 0) ||
        0 > sigaction(SIGQUIT,  &act, 0)) {
        return -1;
    }
    return 0;
}
```
通过**socketpair()**创建一个管道，此管道只在master进程中使用。另外设置master的信号处理handler，当master收到SIGTERM、SIGINT、SIGUSR1、SIGUSR2、SIGCHLD、SIGQUIT这些信号时将调用sig_handler()处理：
```c
static void sig_handler(int signo)
{
    static const char sig_chars[NSIG + 1] = {
        [SIGTERM] = 'T',
        [SIGINT]  = 'I',
        [SIGUSR1] = '1',
        [SIGUSR2] = '2',
        [SIGQUIT] = 'Q',
        [SIGCHLD] = 'C'
    };
    char s;
    ...
    s = sig_chars[signo];
    //将信号通知写入管道sp[1]端
    write(sp[1], &s, sizeof(s));
    ...
}
```

- fpm_sockets_init_main():  创建每个worker pool的socket套接字，启动后worker将监听此socket接收请求。
- fpm_event_init_main():  启动master的事件管理.
启动master的事件管理，fpm实现了一个事件管理器用于管理IO、定时事件，其中IO事件通过kqueue、epoll、poll、select等管理，定时事件就是定时器，一定时间后触发某个事件。



在fpm_init()初始化完成后接下来就是最关键的fpm_run()操作了，此环节将fork子进程，启动进程管理器，另外master进程将不会再返回，只有各worker进程会返回，也就是说fpm_run()之后的操作均是worker进程的。

```c
int fpm_run(int *max_requests)
{
    struct fpm_worker_pool_s *wp;
    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        //调用fpm_children_make() fork子进程
        is_parent = fpm_children_create_initial(wp);
        
        if (!is_parent) {
            goto run_child;
        }
    }
    //master进程将进入event循环，不再往下走
    fpm_event_loop(0);

run_child: //只有worker进程会到这里

    *max_requests = fpm_globals.max_requests;
    return fpm_globals.listening_socket; //返回监听的套接字
}
```

在fork后worker进程返回了监听的套接字继续main()后面的处理，而master将永远阻塞在fpm_event_loop().


#### 2.2 worker-请求处理
fpm_run()执行后将fork出worker进程，worker进程返回main()中继续向下执行，后面的流程就是worker进程不断accept请求，然后执行PHP脚本并返回。整体流程如下：
- **等待请求**： worker进程阻塞在fcgi_accept_request()等待请求。
- **解析请求**： fastcgi请求到达后被worker接收，然后开始接受并解析请求数据，直到request数据完全到达。
- **请求初始化**：执行php_request_startup(), 此阶段会调用每个扩展的PHP_RINI_FUNCTION();
- **编译、执行**：php_execute_script()完成PHP脚本的编译、执行。
- **关闭请求**：请求完成后执行php_request_shutdown()，此阶段会调用每个扩展的：PHP_RSHUTDOWN_FUNCTION()，然后进入步骤(1)等待下一个请求。
```c
int main(int argc, char *argv[])
{
    ...
    fcgi_fd = fpm_run(&max_requests);
    parent = 0;

    //初始化fastcgi请求
    request = fpm_init_request(fcgi_fd);
    
    //worker进程将阻塞在这，等待请求
    while (EXPECTED(fcgi_accept_request(request) >= 0)) {
        SG(server_context) = (void *) request;
        init_request_info();
        
        //请求开始
        if (UNEXPECTED(php_request_startup() == FAILURE)) {
            ...
        }
        ...

        fpm_request_executing();
        //编译、执行PHP脚本
        php_execute_script(&file_handle);
        ...
        //请求结束
        php_request_shutdown((void *) 0);
        ...
    }
    ...
    //worker进程退出
    php_module_shutdown();
    ...
}
```
worker进程一次请求的处理被划分为5个阶段：

- **FPM_REQUEST_ACCEPTING**: 等待请求阶段
- **FPM_REQUEST_READING_HEADERS**: 读取fastcgi请求header阶段
- **FPM_REQUEST_INFO**: 获取请求信息阶段，此阶段是将请求的method、query stirng、request uri等信息保存到各worker进程的fpm_scoreboard_proc_s结构中，此操作需要加锁，因为master进程也会操作此结构
- **FPM_REQUEST_EXECUTING**: 执行请求阶段
- **FPM_REQUEST_END**: 没有使用
- **FPM_REQUEST_FINISHED**: 请求处理完成  
worker处理到各个阶段时将会把当前阶段更新到==fpm_scoreboard_proc_s->request_stage==，master进程正是通过这个标识判断worker进程是否空闲的。

#### 2.3 master-进程管理
<u>master进程管理woker进程管理方式:</u>

- **static**  
worker进程数固定不变。在启动时master按照pm.max_children配置fork出相应数量的worker进程。
- **dynamic**  
动态进程管理。 
    - 首先fpm启动时按照pm.start_servers初始化一定数量的worker（*默认情况为：pm.min_spare_servers + (max_spare_servers - min_spare_servers) / 2*）。
    - 运行期间若master发现空闲的worker数低于pm.min_spare_servers(最小空闲数)配置数（请求较多，worker处理不过来）则会fork进程，但总worker数不能超过pm.max_children(最大进程数)；
    - 若空闲worker数超过pm.max_spare_servers(最大空闲数)（空闲worker数过多），则kill掉一些wokrer，避免占用资源过多。
```
pm.start_servers: 初始worker数
pm.min_spare_servers: 最小空闲worker数量
pm.max_spare_servers: 最大空闲worker数量
pm.max_children: 最大worker数
```
- **ondemand**  
在启动时不分配worker进程，等到有请求了后再通知master进程fork worker进程，总的worker数不超过pm.max_children，处理完成后worker进程不会立即退出，当空闲时间超过pm.process_idle_timeout后再退出。
```
pm.max_children: 最大worker数
pm.process_idle_timeout: 空闲超时时间
```



master整体的处理，其进程管理主要依赖注册的几个事件：

```c
void fpm_event_loop(int err)
{
    //创建一个io read的监听事件，这里监听的就是在fpm_init()阶段中通过socketpair()创建管道sp[0]
    //当sp[0]可读时将回调fpm_got_signal()
    fpm_event_set(&signal_fd_event, fpm_signals_get_fd(), FPM_EV_READ, &fpm_got_signal, NULL);
    fpm_event_add(&signal_fd_event, 0);

    //如果在php-fpm.conf配置了request_terminate_timeout则启动心跳检查
    if (fpm_globals.heartbeat > 0) {
        fpm_pctl_heartbeat(NULL, 0, NULL);
    }
    //定时触发进程管理
    fpm_pctl_perform_idle_server_maintenance_heartbeat(NULL, 0, NULL);
    
    //进入事件循环，master进程将阻塞在此
    while (1) {
        ...
        //等待IO事件
        ret = module->wait(fpm_event_queue_fd, timeout);
        ...
        //检查定时器事件
        ...
    }
}
```
##### 2.3.1. sp[1]管道可读事件：
在fpm_init()阶段master曾创建了一个全双工的管道：`sp`，然后在这里创建了一个sp[0]可读的事件，当sp[0]可读时将交由fpm_got_signal()处理，向sp[1]写数据时sp[0]才会可读，那么什么时机会向sp[1]写数据呢？前面已经提到了：当master收到注册的那几种信号时会写入sp[1]端，这个时候将触发sp[0]可读事件。
![master_event_1](https://note.youdao.com/yws/api/personal/file/0A7863804BB64BAF9886A6C51074E0BD?method=download&shareKey=8cf19b2b30511a2cc2fe04d4dfb8b2ec)
信号用途：

- SIGINT/SIGTERM/SIGQUIT:**退出fpm**，在master收到退出信号后将向所有的worker进程发送退出信号，然后master退出.
- SIGUSR1:**重新加载日志文件**，生产环境中通常会对日志进行切割，切割后会生成一个新的日志文件，如果fpm不重新加载将无法继续写入日志，这个时候就需要向master发送一个USR1的信号
- SIGUSR2:**重启fpm**，首先master也是会向所有的worker进程发送退出信号，然后master会调用execvp()重新启动fpm，最后旧的master退出
- SIGCHLD:这个信号是子进程退出时操作系统发送给父进程的，子进程退出时，内核将子进程置为僵尸状态，这个进程称为僵尸进程，它只保留最小的一些内核数据结构，以便父进程查询子进程的退出状态，只有当父进程调用wait或者waitpid函数查询子进程退出状态后子进程才告终止，fpm中当worker进程因为异常原因(比如coredump了)退出而非master主动杀掉时master将受到此信号，这个时候父进程将调用waitpid()查下子进程的退出，然后检查下是不是需要重新fork新的worker

##### 2.3.2. fpm_pctl_perform_idle_server_maintenance_heartbeat():
这是进程管理实现的主要事件，master启动了一个定时器，每隔**1s**触发一次，主要用于dynamic、ondemand模式下的worker管理，master会定时检查各worker pool的worker进程数，通过此定时器实现worker数量的控制

```
static void fpm_pctl_perform_idle_server_maintenance(struct timeval *now)
{
    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        struct fpm_child_s *last_idle_child = NULL; //空闲时间最久的worker
        int idle = 0; //空闲worker数
        int active = 0; //忙碌worker数
        
        for (child = wp->children; child; child = child->next) {
            //根据worker进程的fpm_scoreboard_proc_s->request_stage判断
            if (fpm_request_is_idle(child)) {
                //找空闲时间最久的worker
                ...
                idle++;
            }else{
                active++;
            }
        }
        ...
        //ondemand模式
        if (wp->config->pm == PM_STYLE_ONDEMAND) {
            if (!last_idle_child) continue;

            fpm_request_last_activity(last_idle_child, &last);
            fpm_clock_get(&now);
            if (last.tv_sec < now.tv_sec - wp->config->pm_process_idle_timeout) {
                //如果空闲时间最长的worker空闲时间超过了process_idle_timeout则杀掉该worker
                last_idle_child->idle_kill = 1;
                fpm_pctl_kill(last_idle_child->pid, FPM_PCTL_QUIT);
            } 
            continue;
        }
        //dynamic
        if (wp->config->pm != PM_STYLE_DYNAMIC) continue;
        if (idle > wp->config->pm_max_spare_servers && last_idle_child) {
            //空闲worker太多了，杀掉
            last_idle_child->idle_kill = 1;
            fpm_pctl_kill(last_idle_child->pid, FPM_PCTL_QUIT);
            wp->idle_spawn_rate = 1;
            continue;
        }
        if (idle < wp->config->pm_min_spare_servers) {
            //空闲worker太少了，如果总worker数未达到max数则fork
            ...
        }
    }
}
```
##### 2.3.3 fpm_pctl_heartbeat():
这个事件是用于限制worker处理单个请求最大耗时的，php-fpm.conf中有一个request_terminate_timeout的配置项，如果worker处理一个请求的总时长超过了这个值那么master将会向此worker进程发送kill -TERM信号杀掉worker进程，此配置单位为秒，默认值为0表示关闭此机制，另外fpm打印的slow log也是在这里完成的。
```
static void fpm_pctl_check_request_timeout(struct timeval *now)
{   
    struct fpm_worker_pool_s *wp;

    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        int terminate_timeout = wp->config->request_terminate_timeout;
        int slowlog_timeout = wp->config->request_slowlog_timeout;
        struct fpm_child_s *child;

        if (terminate_timeout || slowlog_timeout) { 
            for (child = wp->children; child; child = child->next) {
                //检查当前当前worker处理的请求是否超时
                fpm_request_check_timed_out(child, now, terminate_timeout, slowlog_timeout);
            }
        }
    }
}
```
注：*ondemand模式下master监听的新请求到达的事件，因为ondemand模式下fpm启动时是不会预创建worker的，有请求时才会生成子进程，所以请求到达时需要通知master进程，这个事件是在fpm_children_create_initial()时注册的，事件处理函数为fpm_pctl_on_socket_accept()。*




---
参考文档：  
1、php内核剖析： http://www.php.cn/manual/view/32905.html  
2、php7-integernal：https://github.com/pangudashu/php7-internal