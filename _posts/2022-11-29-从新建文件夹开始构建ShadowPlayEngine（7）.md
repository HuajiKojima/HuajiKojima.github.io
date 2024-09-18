---
layout:     post                    # 使用的布局（不需要改）
title:      从新建文件夹开始构建ShadowPlay Engine（7）               # 标题 
subtitle:   多线程驾到         #副标题
date:       2022-11-29             # 时间
author:     zyl                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - UtopiaEngine
    - 引擎开发日志
---

## 序
好久不见，ShadowPlay引擎系列的更新拖了好久。之前一直在忙赶毕业设计，毕业设计完成后又去找实习、顺利毕业后工作896，连周六都没得。不过好消息是，因为毕业设计已经完成并且这个引擎也是我的毕业设计题目。所以如果不出意外的话ShadowPlay系列可以保证一个稳定的周期来进行更新。

在“Unity内存模型原理”一期博文中，引擎中的内存模型是被导师重点批斗的对象，但实际上话没说全，之前的线程管理单元也是“一塌糊涂（导师原话）”。在经过了夜以继日的改进后，新的多线程部分已经完成并集成进引擎内，且听我娓娓道来。

## 一、现有多线程单元存在的问题

在开始之前，让我们具体回顾一下之前的多线程单元，我们真正管理的对象是std::thread，而非计算机内的线程资源，并且实际上，std::thread对象在绑定的函数运行完后就已经没有太大用处了。我们保留的仅仅是一个空壳。那么之前为什么可以运行新的函数呢？那是因为每次申请相当于是做了一个新的线程申请。并没有起到真正的掌握现有线程的职责。也就是说，我们要保证线程内执行的方法必须要同时承担运行任务以及线程休眠。在实现上也就是不许在我们允许关闭此线程之前自动结束线程上的方法。其实和内存管理有异曲同工之处，即我一直占用着部分资源并且让操作系统认为这部分资源一直处于活跃状态并将其保持在用户态的调用中，这样我就可以想什么时候用这些资源就什么时候用，不用考虑当前计算机的繁忙状态。因为申请资源是在初始化就已经进行了的，能进行到此处就说明已经拿到了资源，这部分资源就为我所用。之前的错误就在于实际上std::thread只有在绑定函数运行时才能保持线程被用户态使用。

## 二、线程池实现——任务队列

首先是多线程池如何处理来自引擎交给其的任务，因为线程池所能处理的任务毕竟是有限的，单个线程同时只能执行一条任务。那么线程池同一时刻所能运行的任务数量就是（线程数*1）。但如果一次性一口气提交了多于可处理数量的任务，就要把多出的任务暂时放入一个缓冲区搁置起来。那么这个缓冲区就是目前需要实现的基础：任务队列。

在这里，任务队列可以使用标准库的std::queue实现基础数据结构。我们需要做的就是要根据现实线程池的任务队列运作情况来对std::queue内部的数据以及读取操作和队列操作的设计与实现。首先，由于标准库提供的线程类std::thread仅支持函数的传入，所以std::queue内需要存储的数据是函数类型，接下来是队列的操作：推入和排出。如下所示：

```C++
/**
 * 返回队列是否为空的查询结果
 */
bool QueueIsEmpty()
{
    // 上锁，防止查询时有其他进程进行修改
    std::unique_lock<std::mutex> lck(queueMutex);
    return queueData.empty();
}
/**
 * 返回队列大小
 */
int Size()
{
    // 上锁，防止查询时有其他进程进行修改
    // 虽说此时没有涉及到队列的推入与排除，但不保证别的线程在此时没干这种事情。
    // 所以上锁还是有必要的。
    // 这其实引出一个问题：数据的时效性仅在这个函数的栈生命周期内有效。
    std::unique_lock<std::mutex> lck(queueMutex);
    return queueData.size();
}
/**
 * 数据入队，其中参数emplaceData为输入参数
 */
void Emplace(T& emplaceData)
{
    // 上锁，防止查询时有其他进程进行修改
    std::unique_lock<std::mutex> lck(queueMutex);
    queueData.emplace(emplaceData);
}
/**
 * 数据出队，其中参数popData为输出参数，存储弹出的数据的引用
 */
bool Pop(T& popData)
{
    // 上锁，防止查询时有其他进程进行修改
    std::unique_lock<std::mutex> lck(queueMutex);
    if (queueData.empty())
    {
        return false;
    }
    // 取队列第一个元素并进行右值引用
    popData = std::move(queueData.front());
    // 弹出队列中的第一个元素
    queueData.pop();
    return true;
}
```

这里没有什么太多技术含量，唯一需要说明的是这个任务队列类有两个私有成员：队列数据以及线程互斥锁。其他的就没有什么说明的必要，有不理解的可以参阅注释。

## 三、线程池实现——线程池管理器

接下来我们就有依据创建我们的线程池了。首先线程池可以根据需要改变管理的线程数量，不过仅限在初始化的时候。其次，在初始化的时候我们就需要向系统申请并占用足够数量的线程资源。那么我们可以初步得到线程池的定义：

```C++
// 线程池管理器的声明
class ThreadPool
{
public:
    ThreadPool(const uint32_t threadCount);
    ThreadPool() = delete;
    ThreadPool(const ThreadPool&) = delete;
    ThreadPool(ThreadPool&&) = delete;
    ThreadPool operator= (const ThreadPool&) = delete;
    ThreadPool operator= (ThreadPool&&) = delete;
    ~ThreadPool();
    // 线程池初始化
    void Init();
    // 线程池结束运行
    void Shutdown();

private:
    bool poolShutdown;
    ThreadSafeQueue<std::function<void()>> taskQueue;          // 线程池任务队列
    std::vector<std::thread> workingThreads;                   // 工作线程
    std::mutex conditionMutex;
    std::condition_variable conditionLck;
}
```

在这里，删除掉了默认构造、复制构造、移动构造以及赋值符号重载。由于线程池管理器有且仅有一个并归主线程管理。所以以上构造并没有存在的意义，并且还可能会造成线程调度问题。所以此处删除。

初始化函数才是真正的线程池初始化步骤的发生部分，这里做了构造函数与初始化步骤的分离。线程池结束运行也一致。首先从初始化部分看起：构造函数只用于填入待管理线程数并创建相应长度的线程数组以及打开线程池的开关即可。初始化部分则是向操作系统申请线程并持续保持线程活跃。

接下来是任务提交，线程并不能直接受理我们提交的任务，所以我们要向任务队列里提交，线程池中的线程从任务队列里获取任务并执行。那么我们可以这么写：

```C++
template<typename FT, typename... Args>
auto SubmitTask(FT&& func, Args &&... arg) -> std::future<decltype(func(arg...))>
{
    // 绑定任务函数体，使用std::function保证可填入的类型
    std::function<decltype(func(arg...))()> taskFunc = 
        std::bind(std::forward<FT>(func), std::forward<Args>(arg)...);
}
```

这里又一次使用了C++11中的后置返回值特性，一个新的问题浮上水面：因为执行的任务（函数）类型在任务队列初始化时就已经确定了，是void（）。也就是说不支持任何参数填入，也不支持任何返回值。这时候该怎么办？

问题不大，我们可以为参数以及函数对象打一个包：

```C++
auto taskPointer = std::make_shared<std::packaged_task<decltype(func(arg...))()>>(taskFunc);
```

C++甚至专门为我们提供了这么一个模板类：std::packaged_task。它可以将我们传入的函数对象、参数列表固定打包。它的执行和std::function<void()>以及void(*)()的函数指针如出一辙。但它可以返回一个是std::future对象，我们可以通过这个future对象获取到真实函数运行完成时的返回值。

那么任务提交方法就可以完善了：

```C++
template<typename FT, typename... Args>
auto SubmitTask(FT&& func, Args &&... arg) -> std::future<decltype(func(arg...))>
{
    // 绑定任务函数体，使用std::function保证可填入的类型
    std::function<decltype(func(arg...))()> taskFunc = 
        std::bin(std::forward<FT>(func), std::forward<Args>(arg)...);

    auto taskPointer = std::make_shared<std::packaged_task<decltyp(func(arg...))()>>(taskFunc);

    // 创建任务代理，保证任务队列可以使用固定的模板格式
    std::function<void()> taskFuncSurrogate = [taskPointer]()
    {
        (*taskPointer)();
    };

    // 任务入队
    taskQueue.Emplace(taskFuncSurrogate);
    // 随机叫醒一个空闲线程
    conditionLck.notify_one();
    // 返回本任务的std::future对象
    return taskPointer->get_future();
}
```

我们可以通过lambda表达式创建一个任务代理并将其存入std::function<void()>函数对象。在这一切都完成后，程序将其推入队列并通过condition唤醒一条正在等待的线程。当然如果没有休眠的线程则什么都不做就是了。同时，此函数返回一个future对象。用以向调用方提供任务返回值。

接下来就是线程内的操作了，在序言部分有说过，线程池需要让线程资源在操作系统看来是一直保持占用且活跃状态的。那么也就是说我们需要将等待队列任务与执行任务放入同一个线程执行方法内。这样我们可以定义一个线程执行类，专门用来做以上的工作。声明如下：

```C++
typedef uint64_t ThreadID;

class ThreadExecute
{
public:
    ThreadExecute(ThreadPool* pool, const ThreadID executeID);
    ~ThreadExecute();
    void operator() ();
private:
    ThreadID id;
    ThreadPool* poolTgt;
};
```

由于原生的std::thread线程对象并不知道线程池的存在。所以需要将线程池实例指针以及线程编号传入私有成员，在调试过程中这里存储的线程编号就很有用了。真正执行任务并查询新任务的逻辑在括号运算符重载函数内执行。std::thread可以通过填入类实例的括号运算符重载来运行其内的逻辑。如下：

```C++
void ThreadPool::ThreadExecute::operator()()
{
    // 任务函数对象
    std::function<void()> taskFunction;
    // 查询是否有新的任务进入任务队列
    bool isPopList;
    while (!(poolTgt->poolShutdown))
    {
        // 这里的代码块是为了隔离任务运行与任务查询，因为任务查询的操作是上锁的。
        // 总不可能任务执行的时候也上锁吧，那不和单线程运行没区别了吗（笑）
        {
            std::unique_lock<std::mutex> queueAskLck(poolTgt->conditionMutex);
            if (poolTgt->taskQueue.QueueIsEmpty())
            {
                // 如果没有待执行任务，那么就让线程进入等待
                // 这里的等待会通过提交任务函数那里的notify方法唤醒
                poolTgt->conditionLck.wait(queueAskLck);
            }
            // 告知接下来的步骤线程处在等待中所以不执行任务
            isPopList = poolTgt->taskQueue.Pop(taskFunction);
        }
        if (isPopList)
        {
            // 若有待解决任务则执行
            taskFunction();
        }
    }
}
```

由于这个执行类只有线程管理器知道其状态。所以可以将其声明放入线程池内。以上函数如有不懂的地方可以参阅注释。

那么接下来我们就可以得到线程池管理器的完整声明了：

```C++
// 线程池管理器的声明
    class ThreadPool
    {
    public:
        // 输入参数为创建的可供调用线程总数
        // 同时，由于线程池负责引擎全局的线程管理，单例模式下不存在复制构造以及移动构造的情况
        ThreadPool(const uint32_t threadCount);

        ThreadPool() = delete;

        ThreadPool(const ThreadPool&) = delete;

        ThreadPool(ThreadPool&&) = delete;

        ThreadPool operator= (const ThreadPool&) = delete;

        ThreadPool operator= (ThreadPool&&) = delete;

        ~ThreadPool();

        // 线程池初始化
        void Init();

        // 线程池结束运行
        void Shutdown();

        // 线程执行任务提交
        template<typename FT, typename... Args>
        auto SubmitTask(FT&& func, Args &&... arg) -> std::future<decltype(func(arg...))>
        {
            // 绑定任务函数体，使用std::function保证可填入的类型
            std::function<decltype(func(arg...))()> taskFunc = 
                std::bind(std::forward<FT>(func), std::forward<Args>(arg)...);

            auto taskPointer = std::make_shared<std::packaged_task<decltype(func(arg...))()>>(taskFunc);

            // 创建任务代理，保证任务队列可以使用固定的模板格式
            std::function<void()> taskFuncSurrogate = [taskPointer]()
            {
                (*taskPointer)();
            };

            // 任务入队
            taskQueue.Emplace(taskFuncSurrogate);
            // 随机叫醒一个空闲线程
            conditionLck.notify_one();
            // 返回本任务的std::future对象
            return taskPointer->get_future();
        }
    private:
        /**
         * 标准库线程类所需的执行体类定义
         * 由于标准库内的执行体执行完毕后线程资源被释放，与线程池定义不符
         * 所以本线程池将线程任务执行以及查找任务的流程均放入同一执行体内
         */
        class ThreadExecute
        {
        public:
            ThreadExecute(ThreadPool* pool, const ThreadID executeID);
            ~ThreadExecute();

            void operator() ();

        private:
            ThreadID id;
            ThreadPool* poolTgt;
        };

        bool poolShutdown;                                         // 线程池开关
        ThreadSafeQueue<std::function<void()>> taskQueue;          // 线程池任务队列
        std::vector<std::thread> workingThreads;                   // 工作线程
        std::mutex conditionMutex;                                 // 由执行体进行调用
        std::condition_variable conditionLck;                      // 环境变量锁
    };
```

以上。

## 结语

本次说明了ShadowPlay Engine内的线程池实现思路以及方法，不过目前并没有集成进引擎（只是本章内容所说，真实项目里已经集成），其调用，调试会在之后的更新中一一道来。毕竟这一期的知识密度也很高，希望给各位一个消化空隙。给下一期打个预告：还是多线程部分。不过会集成一种新的多线程工作模式，即任务切片。下期见！

![知识共享许可协议](https://i.loli.net/2021/05/21/FDg2VLNJhyT7ZAE.png)

本作品采用[知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](http://creativecommons.org/licenses/by-nc-sa/4.0/)进行过许可

