---
layout:     post                    # 使用的布局（不需要改）
title:      从新建文件夹开始构建ShadowPlay Engine（5）               # 标题 
subtitle:   内存戏法.         #副标题
date:       2021-08-29             # 时间
author:     zyl                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - UtopiaEngine
    - 引擎开发日志
---

## 本篇序言

从本篇开始，我们要开始构建引擎核心中的系统组件部分，广义上讲其实我们从开始到现在一直都是在构建引擎核心中的系统部分，但严格的定义中系统组件大概有这么几个：内存管理，线程管理，文件管理，时间系统，特殊格式文件处理（比如XML，json文件等）。接下来的文章更新间隔可能会长一些，说不定哪天就断更了。诶嘿。

正如在上一篇文章中我所承诺的那样，接下来我会用大概3至4篇博文的长度来描述本引擎线程管理和内存管理部分，这两个部分同时也是系统组件中不怎么容易理解并且工程量最大的两个模块。我会尽量用通俗易懂的方式描述，希望可以为各位的游戏引擎开发提供帮助。

## 1. 内存管理（第一部分，理论）

这是我们的引擎开发到目前所面临的最复杂的一个单元，所以，我会在我们的引擎项目之外创建一个新项目用来编码与测试我们的内存管理单元与线程管理单元。等我们的以上两个单元完全没有什么问题时，我们再将它们迁移回我们的引擎项目当中。这两个单元说是很复杂，其实也不怎么复杂（听君一席话，如听一席话），至少比《雷神之锤Ⅲ》的那个求平方根倒数的“WTF”函数要好理解得多。那么让我们开始吧！

还记得上篇文章中我所讲到的我们一直在进行的“危险行为”么？盲目地使用大量的new以及delete而不加管理很容易造成内存泄漏，空指针调用，野指针等问题，这只是其一。其二，直接使用new分配会浪费大量的运行时间，为什么这么说？如果各位有使用虚幻或者unity的内存监视的经验的话应该可以发现小内存的分配与释放是最频繁的，比如游戏内事件对象，纹理，AI-Controller对象或者Shader对象等，这些小内存大致都不超过32千字节（实际上32KB大小的内存分配也不怎么频繁，这里其实是取经验数字）。而new与delete进行分配与释放小内存空间的操作所消耗的时间成本可是非常大的。所以我们为什么不在引擎初始化前就在内存中专门划分一块区域用来分配给这些小内存的相关操作？也就是说，小于某个大小标准的内存分配要求我们可以在我们的专门区域为其分配内存空间，也就是将这部分小内存的操作权从操作系统交到我们引擎的手上，而大于这个标准的我们直接为其分配内存。这样可以在可接受范围内的内存浪费的情况下保证引擎运行的高效性。

其实我上面描述的这个算法就是虚幻引擎3的内存管理办法。当然在细节上，我最终在引擎中的实现和虚幻3是不同的，但是大致思路一样。然而这个分配算法由于其精妙性也导致了它的理解门槛有些高，所以希望各位有一定的计算机组成原理以及操作系统的基础，没有的话也没太大关系，我会对一些概念做详细说明，涉及的相关知识点不是很多，所以还请不用担心。

就像我在上面提到过的，在开发调试阶段我们需要内存监视来告诉我们我们的内存占用、内存对齐情况、内存块分配、内存块释放以及内存块大小等信息。而语言层面自带的内存分配可并没有这方面的接口供我们调用，所以我们就必须要自己组织内存管理的数据结构，保证在使用操作系统提供的API时可以跟踪到具体位置。说的这么危言耸听，但其实很简单，在开发调式模式下，我们的内存管理并不需要遵循我上述说到的快速分配算法，我们主要是为了让游戏开发人员在开发过程中可以通过内存监视追踪到出问题的地方，所以我们可以这样去设计在开发调试阶段的内存管理器：

就像上一篇文章中我们构建渲染链的设想一样，引擎是不知道你会申请分配多少或释放多少内存，对于B/S的管理系统来说服务器使用一个线性Pool以及排队等候就可以解决问题，但游戏引擎的实时性要高于B/S的管理系统，游戏玩家可不希望因为预留线性空间不足导致只扣矿石而不生产单位，从而导致贻误战机。目前比较经济的一个方法也就是使用链表来管理这些内存块。

所以，该怎么做？

为了避开系统的自动分配从而导致很难跟踪内存（虽说各种IDE或者操作系统也提供了一大堆的内存跟踪管理的工具以及插件，不过想要找到自己申请分配的那块内存，不多下点功夫还是很难找的到的，多数游戏开发人员并不想将大量的时间与精力浪费到一连串的16进制数里面，而且游戏引擎是一套工具集，它有义务为开发人员提供更直观的内存监视结果），我们有两条路可走：C语言的malloc或者是汇编。不过由于我们的引擎只能在x64环境下运行（这是当初构建项目时已经设计好的），也就导致了我们无法直接在代码文件中使用嵌入式汇编语句“__asm”。还有，因为本人技术不过关，win32的一些指令到了x64就要重新考虑了。而且malloc的分配后的内存结构也便于理解，所以我们选择malloc以及free，比起new以及delete更加自由也更加基础一些。

而每个链表的结点我们可以这样设计：

![pic1.png](https://i.loli.net/2021/08/29/53SJIsutUPfGwey.png)

这次我依旧使用了使用代理类MemoryBlock，但代理类与被代理对象并不是通过指针相联系了，而是将它们通过一段连续的内存联系起来，因为这次我们的被代理对象就是内存块啊（笑），而且代理类没有义务也没有权限去了解被代理内存中对象的类型，这种联系方式使得内存在释放时也会更有效率一些。

好了，大致的设计构想我们已经总结完毕，接下来让我们考虑一些小细节问题，假设我们最终写成的分配器以及释放器的声明如下所示：

```C++
void* memAllocate(unsigned long long _udlLength, bool _bIsArray);	// 分配器
void memDeallocate(void* _pBlock, bool _bIsArray);					// 释放器
```

看起来并没有什么问题，很好解释：分配器需要的参数是待分配内存的长度以及是否是数组的条件值，释放器需要的参数是待释放的指针地址以及是否是数组的条件值。也就是说在每次分配内存时我们都需要向里面填入相应的参数，这就是对比new来说一个稍微麻烦的一点了，既然new操作符有这么好的特性，那我们就把它用在我们的分配器上，但有聪明的同学会立马提出质疑：你不是说过不能随便用new以及delete关键字吗？那么，我们是否可以通过重载这两个关键字得到我们想要的效果？幸运的是，C++支持对new进行运算符重载，所以，我们的工作立马就容易得多。我们可以在引擎的作用域内重载new以及delete运算符，然后在重载函数体里调用我们的分配器与释放器。用这种“偷天换日”的方法在不影响游戏开发人员的开发效率下完成引擎对内存的管理工作。

也就是说，我们可以这么去写：

```C++
// 负责单个内存块分配工作
void* operator new(unsigned long long _udlLength)
{
   return memAllocate(_udlLength, false);
}
// 负责数组分配工作
void* operator new[](unsigned long long _udlLength)
{
   return memAllocate(_udlLength, true);
}
// 负责单个内存块释放工作
void operator delete(void* _pBlock)
{
   return memDeallocate(_pBlock, false);
}
// 负责数组释放工作
void operator delete[](void* _pBlock)
{
   return memDeallocate(_pBlock, true);
}
```

在后面的第二部分，我们会开始着手构建在调试环境下的内存分配与管理器。

## 2. 线程管理单元（第一部分）

我们先不着急接着往下进行我们的内存管理部分，在继续之前请容大家与我解决一些小障碍。就像我在构建渲染核心时候说的一样，OpenGL的“初始化——渲染循环——释放”是一个典型的状态机模型，也就是说它是遵循单线程模式的。而且引擎的所有非初始化类型的处理语句都必须运行在渲染循环中以确保实时更新。这看起来并没有什么问题，但问题也正出在这里，过长的处理步骤必然会占用更多的处理器时间，也就是说循环体中的一次循环执行时间会更长，这样必然导致游戏帧数低下，所以我们必须要将某些对实时性要求不高的处理语句从渲染循环中抽离出来，专门为它们开辟一个或者多个线程，充分利用多核处理器的优势（因为现在也没多少人用单核处理器了），在不影响内容展现的同时，达到引擎运行的高效率。

有两种实现多线程的方法：一种是操作系统提供的API（系统层面），另一种是在C++11中开始提供的thread标准库（语言层面），这里我们选择语言层面的实现，举个例子来说明一下两者的差别以及我为什么要选用语言层面的实现：

我们先来看一下Windows启动线程的方法：（摘自Microsoft Docs）

```C++
uintptr_t _beginthreadex( // NATIVE CODE
   void *security,
   unsigned stack_size,
   unsigned ( __stdcall *start_address )( void * ),
   void *arglist,
   unsigned initflag,
   unsigned *thrdaddr
);
```

因为其他的参数目前没有必要去深入，所以挑两个最重要的讲：函数指针start_address以及空指针arglist，start_address也就是我们要在线程中运行的函数，而且Windows在它的参数方面有着严格的定义，即**必须为void\***，空指针arglist就是传参的，Windows只给我们提供了一个参数的预留位置，如果我们想要将更多的参数传入函数内，我们就必须要用到结构体了，这不失为一种巧妙的方法，然而，在对线程需求量大的时候，越积越多的结构体定义除了降低代码可读性外并没有任何好处。而且，我们传入的是指针，也就是说我们可能要面临着更频繁且更复杂的内存分配，虽说操作系统层面的线程管理更加高效，但以这种牺牲掉内存操作时间来换取线程的高运行效率的做法恐恕本人无法接受。当然，它也有优点，比如它拥有对于安全性的操作，线程控制权等等标准库所没有的功能。

接下来我们再看一下标准库启动线程的方法：

```C++
template<class _Func, class..._Args>
std::thread::thread(_Func _f, _Args... _args);
```

这里也就是标准库的一大强项：支持可变参数模板，也就是说我们完全没必要声明为每一个待处理函数设计声明一个新的结构体，直接将参数传入即可，而且相比Windows最大的一个优点就是支持多平台迁移。但标准库的线程操作简单到过于离奇，所以线程相关的操作问题还得我们自己去考虑。

看到这也许各位会问，那我们就直接调用相关层面的API即可，为什么还要大费周章的创建一个线程管理器？正如字面意思，就是为了方便管理线程，如果各位还是有些不理解，还请随我继续接下来的探索：

标准库中是以类的形式抽象线程的，即我们创建一条新线程就是创建了一个新的对象，但由于标准库启动线程是使用上述的构造函数的，也就是说我们可以在任何时间任何地方更改我们线程对象所负责的线程而不必关心当前负责的线程是否结束，听起来就是一个极富灾难性的操作，而且在分配线程后而不及时收回轻者会造成系统资源浪费，重者则直接导致“Abort() has called”。所以我们很有必要为引擎构建一套线程管理单元。

下图便是我为本引擎设计的线程管理单元的大致结构：

![pic2.png](https://i.loli.net/2021/08/29/fuzlU8T35PxXLJI.png)

本线程管理单元由四个部分组成：线程管理器（SPThreadManager，继承自引擎的链表数据结构），线程对象（SPThreadObject），线程ID（SPThread）以及互斥量（SPThreadMtx）。而用户创建新线程时得到的只是线程ID，这一点是从Windows那里学来的，就如Windows的HANDLE一样，用户不能直接通过线程ID来访问线程，他们需要本引擎提供的方法以及他们手里掌握的线程ID来对线程进行操作。这样大大增加了引擎对线程的掌控。互斥量作为对线程相关操作限制的开关存在。

先来介绍一下线程ID，虽说用0~32767作为线程ID完全够用，但总感觉缺少了一点专业气息，显得我是一个“new money”（笑），实际上还有一点考虑，我们希望在后续的引擎功能扩展中加入线程的相关限制，而且我们希望这些限制保存在ID里而不是再开辟一至多块内存用来存放布尔值，这时候一个整型数字可就不能再存放这么多信息了。所以我们使用GUID作为我们引擎的ID，幸运的是，Windows提供了这方面的生成API：coCreateGuid。我们将其稍微的包装一下就可以得到我们的线程ID对象了。声明如下：

```C++
class SPGlobalID
{
public:
    // 构造函数与析构函数，在初始化的时候就生成一个GUID
	SPGlobalID();
	SPGlobalID(const SPGlobalID&);
	~SPGlobalID();

    // 用来做比较用的，主要对比的是GUID值
	bool operator ==(Shadow::SPGlobalID& _right);
	bool operator !=(Shadow::SPGlobalID& _right);
	// 获取相关值，只能用作打印或其他不修改数据的用途
    const GUID& GetWPId() const;
	const std::string& GetSId() const;

    // 设置友元函数，用来在调试信息上输出ID
	friend std::ostream& operator <<(std::ostream& _outObj, SPGlobalID& _id);

private:
	std::string s_ID;
	GUID wp_ID;
};
```

在解决了ID的问题后，我们接下来着手解决线程管理单元，与内存管理以及渲染链一样，引擎除了几个自己用的基础的线程外，根本不知道开发人员在开发游戏或插件时又会申请分配多少个线程，所以，在这里我们还是使用链表来解决问题，在前面讲内存管理的时候我曾说过，频繁的内存分配与释放会消耗大量的CPU时间，而链表则是这其中的常客了。这里我借鉴了一些来自于后端开发者的经验：线程池。也就是说引擎在初始化时会首先创建一定数量的线程对象构成一个线程池，如果分配行为产生，则会首先在线程池中寻找空闲线程对象，线程池内没有空闲线程对象时才会申请一个新的线程对象，这样在一定程度上可以减少CPU时间的浪费，下面是线程管理单元的声明：

```C++
// 正如各位所看到的，我使用typedef来对线程对象以及线程ID做了名称定义
typedef std::thread     SPThreadObject;
typedef SPGlobalID      SPThread;

class SPThreadManager :
	protected LinklistManager<SPThreadObject, SPThread>
{
public:
	SPThreadManager();
	~SPThreadManager();

	// 以下两个方法便是向线程管理单元申请分配线程以及释放线程
    SPThread ApplyThread();
	void TerminateThread(SPThread&);
	
    // 由于我们需要使用可变参数模板，而且这个函数内也并未出现分支判断结构，所以定义在这里
	template <class Fp, class ...Args>
	void ThreadStart(SPThread _thread, Fp _func, Args... _args)
	{
		LinkListNode<SPThreadObject, SPThread>* ln_pointer = this->operator[](_thread);
		ln_pointer->GetContent() = std::thread(_func, _args...);
	}
    // 停止相关线程
    // 但是我们没有理由强制停止，所以这个方法仅仅是调用了一下joinable而已
	void ThreadEnd(SPThread);
private:
    // 这里我用标准库的链表来标记已被占用线程以及空闲线程（线程池）
	std::vector<SPThread> stdv_originalPool;
	std::list<SPThread> stdl_idleThreads;
	std::list<SPThread> stdl_occupiedThreads;
};
```

## 3. 本篇结语

先写到这里，由于内存管理器部分还有许多待调试的地方未完成，所以文章记录也不能很好地进行下去。只有准备万全，才可拿得出手——这是本人的编程守则，也是为了在毕业设计答辩的时候不至于演示翻车（笑）。本次内容不多，但复杂的地方还是有，也希望大家可以慢慢消化，为后面的理解提供基础，俗话也是这么说的：“老鼠拖木锨，大头在后面“。好的，下次见~

![知识共享许可协议](https://i.loli.net/2021/05/21/FDg2VLNJhyT7ZAE.png)

本作品采用[知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](http://creativecommons.org/licenses/by-nc-sa/4.0/)进行过许可
