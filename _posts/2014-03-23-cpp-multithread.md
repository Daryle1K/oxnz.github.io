---
layout: post
title: C++ 多线程编程总结
type: post
categories:
- C++
tags:
- c++
- thread
---

在开发C++程序时，一般在吞吐量、并发、实时性上有较高的要求。
设计C++程序时，总结起来可以从如下几点提高效率:

* 并发
* 异步
* 缓存

<!--more-->

下面将我平常工作中遇到一些问题例举一二，其设计思想无非以上三点。

## Table of Contents

* TOC
{:toc}

## 任务队列

### 以生产者-消费者模型设计任务队列

生产者-消费者模型是人们非常熟悉的模型，比如在某个服务器程序中，当 user 数据被逻辑模块修改后，就产生一个更新数据库的任务（produce），投递给 I/O 模块任务队列，I/O 模块从任务队列中取出任务执行 sql 操作（consume）。

设计通用的任务队列，示例代码如下:

```cpp
void task_queue_t::produce(const task_t& task_) {
    lock_guard_t lock(m_mutex);
    if (m_tasklist->empty()) { //! 条件满足唤醒等待线程
        m_cond.signal();
    }
    m_tasklist->push_back(task_);
}
int task_queue_t::comsume(task_t& task_) {
    lock_guard_t lock(m_mutex);
    while (m_tasklist->empty()) { //! 当没有作业时，就等待直到条件满足被唤醒
        if (false == m_flag) return -1;
        m_cond.wait();
    }
    task_ = m_tasklist->front();
    m_tasklist->pop_front();
    return 0;
}
```

### 1.2    任务队列使用技巧

#### 1.2.1 I/O 与 逻辑分离

比如网络游戏服务器程序中，网络模块收到消息包，投递给逻辑层后立即返回，继续接受下一个消息包。
逻辑线程在一个没有 I/O 操作的环境下运行，以保障实时性。示例：

```cpp
void handle_xx_msg(long uid, const xx_msg_t& msg){
    logic_task_queue->post(boost::bind(&servie_t::proces, uid, msg));
}
```

注意，此模式下为单任务队列，每个任务队列单线程。

#### 1.2.2  并行流水线

上面的只是完成了 I/O 和 cpu 运算的并行，而 cpu 中逻辑操作是串行的。
在某些场合，cpu 逻辑运算部分也可实现并行，如游戏中用户A种菜和B种菜两种操作是完全可以并行的，因为两个操作没有共享数据。最简单的方式是A、B相关的操作被分配到不同的任务队列中。
示例如下：

```cpp
void handle_xx_msg(long uid, const xx_msg_t& msg) {
    logic_task_queue_array[uid % sizeof(logic_task_queue_array)]->post(
        boost::bind(&servie_t::proces, uid, msg));
}
```

注意，此模式下为多任务队列，每个任务队列单线程。

#### 1.2.3 连接池与异步回调

比如逻辑 service 模块需要数据库模块异步载入用户数据，并做后续处理计算。
而数据库模块拥有一个固定连接数的连接池，当执行 SQL 的任务到来时，选择一个空闲的连接，执行 SQL，并把 SQL 通过回调函数传递给逻辑层。
其步骤如下：

1. 预先分配好线程池，每个线程创建一个连接到数据库的连接
1. 为数据库模块创建一个任务队列，所有线程都是这个任务队列的消费者
1. 逻辑层向数据库模块投递 SQL 执行任务，同时传递一个回调函数来接受 SQL 执行结果

示例如下：

```cpp
void db_t:load(long uid_, boost::function<void (user_data_t&) func_){
    //! sql execute, construct user_data_t user
    func_(user)
}
void process_user_data_loaded(user_data_t&){
    //! todo something
}
db_task_queue->post(boost::bind(&db_t:load, uid, func));
```

注意，此模式下为单任务队列，每个任务队列多线程。

## 2. 日志

本文主要讲 C++ 多线程编程，日志系统不是为了提高程序效率，但是在程序调试、运行期排错上，日志是无可替代的工具，相信开发后台程序的朋友都会使用日志。常见的日志使用方式有如下几种：

* 流式
* printf 格式

```cpp
logstream << "start servie time[%d]" << time(0) << " app name[%s]" << app_string.c_str() << endl;
logtrace(LOG_MODULE, "start servie time[%d] app name[%s]", time(0), app_string.c_str());
```

二者各有优缺点，流式是线程安全的，printf 格式格式化字符串会更直接，但缺点是线程不安全。

如果把 app_string.c_str() 换成 app_string （std::string），编译被通过，但是运行期会 crash（如果运气好每次都 crash，运气不好偶尔会 crash）。
我个人钟爱 printf 风格，可以做如下改进：

增加线程安全，利用 C++ 模板的 traits 机制，可以实现线程安全。

示例：

```cpp
template<typename Arg>
void logtrace(const char* module, const char* fmt, Arg arg) {
    boost::format s(fmt);
    f % arg;
}
```

这样，除了标准类型 +std::string 传入其他类型将编译不能通过。
这里只列举了一个参数的例子，可以重载该版本支持更多参数，如果你愿意，可以支持 9 个参数或更多。

为日志增加颜色，在 printf 中加入控制字符，可以再屏幕终端上显示颜色，Linux 下示例：

```c
printf("33[32;49;1m [DONE] 33[39;49;0m")
```

更多颜色方案参见：

http://hi.baidu.com/jiemnij/blog/item/d95df8c28ac2815cb219a80e.html

每个线程启动时，都应该用日志打印该线程负责什么功能。这样，程序跑起来的时候通过top –H – p pid 可以得知那个功能使用cpu的多少。实际上，我的每行日志都会打印线程id，此线程id非pthread_id，而其实是线程对应的系统分配的进程id号。

## 3. 性能监控

尽管已经有很多工具可以分析 c++ 程序运行性能，但是其大部分还是运行在程序 debug 阶段。我们需要一种手段在 debug 和 release 阶段都能监控程序，一方面得知程序瓶颈之所在，一方面尽早发现哪些组件在运行期出现了异常。

通常都是使用 gettimeofday 来计算某个函数开销，可以精确到微妙。
可以利用 C++ 的确定性析构，非常方便的实现获取函数开销的小工具。

示例如下：

```cpp
struct profiler {
    profiler(const char* func_name) {
        gettimeofday(&tv, NULL);
    }
    ~profiler() {
        struct timeval tv2;
        gettimeofday(&tv2, NULL);
        long cost = (tv.tv_sec - tv.tv_sec) * 1000000 + (tv.tv_usec - tv.tv_usec);
        //! post to some manager
    }
    struct timeval tv;
};
#define PROFILER() profiler(__FUNCTION__)
```

cost 应该被投递到性能统计管理器中，该管理器定时讲性能统计数据输出到文件中。

## 4 Lambda 编程

使用 foreach 代替迭代器

很多编程语言已经内建了 foreach，但是 c++ 还没有。所以建议自己在需要遍历容器的地方编写 foreach 函数。

习惯函数式编程的人应该会非常钟情使用 foreach，使用 foreach 的好处多多少少有些，如：

http://www.cnblogs.com/chsword/archive/2007/09/28/910011.html

但主要是编程哲学上层面的。

示例：

```cpp
void user_mgr_t::foreach(boost::function<void (user_t&)> func_) {
    for (iterator it = m_users.begin(); it != m_users.end() ++it) {
        func_(it->second);
    }
}
```

比如要实现 dump 接口，不需要重写关于迭代器的代码

```cpp
void user_mgr_t:dump() {
    struct lambda {
        static void print(user_t& user) {
            //! print(tostring(user);
        }
    };
    this->foreach(lambda::print);
}
```

实际上，上面的代码变通的生成了匿名函数，如果是 c++ 11 标准的编译器，本可以写的更简洁一些：

```cpp
this->foreach([](user_t& user) {});
```

但是我大部分时间编写的程序都要运行在 centos 上，你知道吗它的 gcc 版本是 gcc 4.1.2， 所以大部分时间我都是用变通的方式使用 lambda 函数。

Lambda 函数结合任务队列实现异步

常见的使用任务队列实现异步的代码如下：

```cpp
void service_t:async_update_user(long uid){
    task_queue->post(boost::bind(&service_t:sync_update_user_impl, this, uid));
}
void service_t:sync_update_user_impl(long uid){
    user_t& user = get_user(uid);
    user.update()
}
```

这样做的缺点是，一个接口要响应的写两遍函数，如果一个函数的参数变了，那么另一个参数也要跟着改动。并且代码也不是很美观。

使用 lambda 可以让异步看起来更直观，仿佛就是在接口函数中立刻完成一样。示例代码：

```cpp
void service_t:async_update_user(long uid){
    struct lambda {
        static void update_user_impl(service_t* servie, long uid){
            user_t& user = servie->get_user(uid);
            user.update();
        }
    };
    task_queue->post(boost::bind(&lambda:update_user_impl, this, uid));
}
```

这样当要改动该接口时，直接在该接口内修改代码，非常直观。

## 5. 奇技淫巧

### 利用 shared_ptr 实现 map/reduce

Map/Reduce 的语义是先将任务划分为多个任务，投递到多个 worker 中并发执行，其产生的结果经 reduce 汇总后生成最终的结果。

shared_ptr 的语义是什么呢？当最后一个 shared_ptr 析构时，将会调用托管对象的析构函数。
语义和 map/reduce 过程非常相近。
我们只需自己实现讲请求划分多个任务即可。
示例过程如下：

定义请求托管对象，加入我们需要在10个文件中搜索“oh nice”字符串出现的次数，定义托管结构体如下：

```cpp
struct reducer {
    void set_result(int index, long result) {
        m_result[index] = result;
    }
    ~reducer(){
        long total = 0;
        for (int i = 0; i < sizeof(m_result); ++i){
            total += m_result[i];
        }
        //! post total to somewhere
    }
    long m_result[10];
};
```

定义执行任务的 worker

```cpp
void worker_t:exec(int index_, shared_ptr<reducer> ret) {
    ret->set_result(index, 100);
}
```

将任务分割后，投递给不同的 worker

```cpp
shared_ptr<reducer> ret(new reducer());
int ntask = 10;
for (int i = 0; i < ntask; ++i) {
    task_queue[i]->post(boost::bind(&worker_t:exec, i, ret));
}
```
