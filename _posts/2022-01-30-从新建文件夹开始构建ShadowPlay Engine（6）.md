---
layout:     post                    # 使用的布局（不需要改）
title:      从新建文件夹开始构建ShadowPlay Engine（6）               # 标题 
subtitle:   做成B站视频的话应该很助眠(笑)         #副标题
date:       2022-01-30             # 时间
author:     zyl                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - UtopiaEngine
    - 引擎开发日志
---

## 本篇序言

在经历了为期很长时间的调试以及思维纠错后，我们可以开始实现我们的内存管理模块了，我在前面说过如果各位要继续跟着学习的话可能会需要一定的计算机组成原理和操作系统的知识，不过在莽代码的过程中，我逐渐发现由于本引擎无论是从设计思路上还是代码实现上实在是太过于Rookie。所以秉承着简单到底的精神，我将一部分比较贴近底层的实现交给操作系统自己去完成，我们只要完成相关逻辑就好，所以也就不强制要求大家掌握这些方面的知识。毕竟本人技术力和相关经验也不是炉火纯青，能开发完一个初级的游戏引擎已经是谢天谢地了（笑），同时也感谢看到这里的各位，各位能在本人这极度生草的表达下还能坚持不懈地看到这里，想必这定是不亚于黑魂的极度煎熬和痛苦，本人深感钦佩并表以衷心的感谢。接下来废话不多说，让我们开始今天的内容。

## 1. 内存管理（第二部分，调试模式）

首先让我来填一下上次留下的坑：我们一起完成调试模式下引擎的内存管理。关于内存管理我准备分为两部分去讲，一部分是本节的调试模式，另一部分是接下面两节的Release模式。关于我为什么会这么分其实是有理由的，在游戏开发过程中，开发者会选择调试模式对工程项目内的所有数据进行跟踪调试，我们的内存空间也不例外，比如某个由于未能及时释放内存而导致的崩溃错误是很难通过单步跟踪找出来的，尤其是在代码量巨大的情况下，而且就像我在上一篇说的，开发者并不想面对VS内存调试器里那宛如《大悲咒》一般的一大堆十六进制数，毕竟GamePlay就已经够让他们头秃了。至少我们现在可以做的就是为开发者找到未按规范使用的内存单元并定位出来，并通过Log打印在控制台上好让开发者顺利找到问题所在。

接下来聊一下实现思路：首先，我们的内存管理必须实现“管理”，也就是说，引擎可以找到全局范围内（这里的全局范围就是指游戏进程）的所有动态内存并加以管制，这是其一。其二，我们必须要让内存块预留一定的空间来记住其申请位置。所以我们的大体思路也便明了起来：代理类，也就是说我们可以使用代理类来完成这方面的工作，然后组成一个代理类链表，申请内存的时候就新添加代理结点上去，释放内存的时候就找到相应的代理结点并释放掉。话说的这么好听，具体怎么实现？

还记得我上一篇博文里摆出的那张图吗？没错，就是它：

<img src="https://i.loli.net/2021/08/29/53SJIsutUPfGwey.png" alt="pic1.png"  />

这里的这个代理类并不是我们常识中所说的那个代理类（如果有对代理类概念不熟悉的同学，可以翻看我以前的博客有关于引擎渲染链那一节的内容，那里有对代理类这个概念的描述），常识中的代理类是独立于被代理对象之外的，而这个代理类几乎就是被代理对象自己，原因也很简单，由于指向我们动态申请的空间的指针有可能是任意类型的指针，所以普通代理类中指向单一类型内容的指针就不再适合这种情况了。可能有人会说：我们可以尝试着让引擎中所有的类（甚至包括我们自己设计的类）都继承自同一个最基础的基类不就好了，类似于Java那样。这确实是一个讨巧的办法，但我们不敢保证所有被动态申请的内存是我们或者说是引擎定义的类，毕竟基础数据类型（int、float等）以及标准库的相关类（string、vector等）也可以拿来申请动态内存并交由我们的引擎管理。而且我们这么设计的代理类同时也节约了一部分非必要的内存，毕竟时间复杂度以及空间复杂度在实时渲染里面是完全失灵的，实时渲染只讲究绝对的效率，也就是在算复杂度时常常被我们忽略的常数k。

接下来让我们开始实现：首先我们可以得到调试模式下的内存管理单元的类定义：

```C++
class SPDbgMemManage : public SPMemManage
{
public:
    /**
    * 构造函数，首先构造一个内存管理链的头节点并调用其构造函数，
    * 或许有同学会问：你不是说要内存管理吗，那这里的内存分配怎么算？
    * 这里很好解释，由于内存管理单元本身的一切内存分配与释放流程是可控的，
    * 也就是说，在引擎设计阶段我们就知道它该在哪里做出什么样的内存操作。
    * 所以在这里就没有必要再进行内存管理了。
    */
	SPDbgMemManage()
	{
		mb_blockChain = (SPMemDbgBlock*)malloc(sizeof(SPMemDbgBlock));
		mb_blockChain->SPMemDbgBlock::SPMemDbgBlock();
	}
	~SPDbgMemManage()
	{
        // 析构函数，这里会调用一个TerminateBlock()方法，
        // 不过不用着急，我后面会说它是干什么的。
		TeminateBlock();
        // 释放掉内存管理链头节点所占用的内存，
        // 由于内存管理链里的头节点只负责相关数据的存取，
        // 所以没有必要调用其析构函数，可直接释放。
		free(mb_blockChain);
	}
	
    // 这个方法负责内存分配，它会返回分配得到的空间的指针
    void* memAllocate(MemLength _uiLength, bool _bIsArray);
    // 这个方法负责回收分配出去的内存。
	void memDeallocate(void* _pAddr);
    /** 
    * 为了保证我们的每个被分配出去的内存可被跟踪，我设计了一个数据结构，
    * 叫做CallerLoc，内部实现其实很简单，就是一个字符串类型和一个整数类型，
    * 分别存储分配操作所在的源代码位置以及所在行。
    * 在以后如果发生了由于疏忽导致的分配的内存未被释放的情况，
    * 内存管理单元可以通过每个内存块里的这个数据结构快速定位未被释放内存所在代码的位置，
    * 并将其打印出来提示给开发者。
    */
	void SetCallerLine(void* _pAddr, std::string _sFileDir, unsigned int _uiLine);
    // 搜寻符合条件的内存块，条件就是指向内存单元的指针
	SPMemDbgBlock* SearchBlock(void* p_Addr);

private:
    /**
    * 这就是我们前面提到过的方法，试想一下，
    * 我们的引擎在结束运行前的收尾工作时遇到了一部分开发者分配的内存未被回收，
    * 这时引擎该怎么办？
    * 是直接抛出异常停止运行还是说只是跳出警告但还是按照正常流程进行？
    * 我们的引擎其实可以将其强制释放并报出警告信息提示开发者，
    * 这么做的原因是我们的内存管理单元此时处在调试阶段，
    * 调试阶段可以这么去做，但同时很危险的是，它不会调用被释放内存的析构，
    * 所以这种强制释放内存的操作也只能在调试阶段使用。
    */
	void TeminateBlock();
    // 多线程互斥量
	SPThreadMtx tm_memMtx;
    // 内存管理链
	SPMemDbgBlock* mb_blockChain;
    // 内存管理链上此时拥有的内存块数量
	unsigned int i_blockNum = 0;
};
```

接下来我们的操作就是简单粗暴且“平易近人”的链表操作了，先放出链表节点定义：

```c++
class SPMemDbgBlock
{
public:
    // 这里就是负责链表节点定位的数据结构的Get方法
	CallLoc& GetCallLoc()
	{
		return cl_callerLoc;
	}
private:
    // 构造函数，为了防止这种敏感的对象被肆意篡改，
    // 所以将默认构造函数的可见性设置为private.
	SPMemDbgBlock() :
		mb_previous(nullptr),
		mb_next(nullptr),
		i_memLength(0),
		b_isArray(false)
	{
		// Nothing here, do not waste your time.
	}
    
    // 这句话就表明这个类“一生只爱一个人”，也就是我们的调试内存管理单元
    // 也就只有它才拥有访问这个类的最高权限。
	friend class SPDbgMemManage;
    // 负责定位的数据结构对象
	CallLoc cl_callerLoc;
    // 我也不知道我究竟是为啥会做一个双向链表出来，
    // 后期调试的时候在这里栽了一个大跟头，
    // 其实可以用单向链表实现，实现简单同时也相比于双向链表节约出一个指针来
	SPMemDbgBlock* mb_previous;
	SPMemDbgBlock* mb_next;
    // 本内存结点所代理的内存块的大小
	MemLength i_memLength;
    // 判定内存块的性质（顺序表还是单个空间）
	bool b_isArray;
};
```

好了，在做完了以上两个重要组成部分的定义之后，我们接下来开始挑最重要的两个部分来谈谈：Allocate以及Deallocate。我会放出它们实现的方法，具体每一步的意义与解释我会以注释的形式标明：

Allocate：

```C++
void* SPDbgMemManage::memAllocate(MemLength _uiLength, bool _bIsArray)
{
    /** 
    * 试想这样一个问题：目前为止我们所考虑的许多情况都是基于单线程的，
    * 但是如果此时渲染线程和场景读取线程同时需要进行内存的分配请求，我们的引擎该怎么办？
    * 确实，我们可以在两个线程内同时运行这个方法，同时为其分配内存，
    * 但你需要知道的一个前提是，我们不能预测引擎在运行时会同时开辟几个线程，
    * 这也就意味着我们不能在内存管理器中留足够的链表头节点来专门为每一个线程进行内存管理。
    * 当然目前有些引擎确实拥有这方面的技术，但至少我们的引擎暂时是建立在这方面不可知的条件下，
    * 即表明在上述情况中我们的内存管理单元极有可能因为多线程的情况而导致失去一部分内存的控制权。
    * 这种情况所造成的后果是灾难性的，上一个线程的内存块可能会被下一个线程覆盖掉，
    * 所以我们需要在这里为这个函数添加一个互斥锁来保证这种情况不会发生。
    */
	while (!this->tm_memMtx.ThreadLock());
    // 这里获得内存块代理类（内存管理链的节点）的长度。
	MemLength ui_Block = sizeof(SPMemDbgBlock);
    // 分配空间，分配的空间长度是连续的，组成也就是“节点 + 内存块”。
	void* m_TempMem = (void*)malloc(_uiLength + sizeof(SPMemDbgBlock));
    // 相信这种分支判断的意义大家在C语言以及数据结构中已经了如指掌了，不再赘述
	if (m_TempMem)
	{
         // 接下来我们需要一个内存块类型的指针帮我们代为完成任务
         // 这么做的目的是，void* 类型的指针无法参与任何操作与运算，
         // 它只有传递地址内容的作用。
		SPMemDbgBlock* mb_TempBlock = (SPMemDbgBlock*)m_TempMem;
         // 在得到新空间后按照常理，我们需要手动调用节点的构造函数。
		mb_TempBlock->SPMemDbgBlock::SPMemDbgBlock();
         // 接下来的几步操作就是纯粹的链表操作了，这里不做赘述。
		mb_TempBlock->b_isArray = _bIsArray;
		mb_TempBlock->i_memLength = _uiLength;
		mb_TempBlock->mb_next = this->mb_blockChain->mb_next;
		mb_TempBlock->mb_previous = this->mb_blockChain;
		this->mb_blockChain->mb_next = mb_TempBlock;
		if (mb_TempBlock->mb_next != nullptr)
		{
			mb_TempBlock->mb_next->mb_previous = mb_TempBlock;
		}
         // 当然，请记住，在一切操作做完后还请把互斥锁打开，
         // 以便于其他线程访问此函数
		this->tm_memMtx.ThreadUnlock();
        /**
        * 这里有点说头，我来和大家仔细谈一谈：
        * 这里的意思是，我先让目标指针跳跃一个位置（而并不是跳跃一个字节或者是一位），
        * 而这一个位置映射的长度就是链表节点类的长度，
        * 至于为什么之后我需要用强制类型转换（void*）呢？
        * 一个原因是为了迎合函数定义里面 void* 的返回值，
        * 还有一个最重要的原因是：如果我并没有做强制类型转换，C++是会按照原本指针的类型进行内存截断。
        * 这里的内存截断并不是说真的截掉并将截掉的部分归还，而是说C++会按照原本的类型去组织这段内存，
        * 当我们申请分配的内存大小小于链表节点类时，这段内存的指针却还是链表节点类的指针，
        * 那么就会有一部分原本不属于我们所申请分配的内存里的内容被篡改（因为它们连着我们申请分配的内存单元）
        * 而这部分内存里存的什么内容没人知道，这就要看具体的操作系统如何组织内存了，
        * 这也表明，我们有可能更改了操作系统自身的内存数据也说不定，这种结果是灾难性的（育碧直呼内行）。
        */
		return (void*)(mb_TempBlock + 1);
	}
    // 在分配内存失败时需要做的操作。
	this->tm_memMtx.ThreadUnlock();
	return nullptr;
}
```

Deallocate：

```C++
void SPDbgMemManage::memDeallocate(void* _pAddr)
{
    // 与Allocate一致，这里不再说明
	while (!this->tm_memMtx.ThreadLock());
    // 这里我们会使用本类里的方法SearchBlock去寻找管理相关内存的节点。
	SPMemDbgBlock* mb_tgtBlock = this->SearchBlock(_pAddr);
    // 如果引擎要是没有找到，那就说明已经释放掉了或者说根本就不存在，
    // 不过我们并不会像CL或者GCC那样直接Abort，
    // 而是弹出警告告诉开发者，指针所指向的内存不存在，
    // 接下来究竟是终止运行还是无伤大雅地继续运行那就看开发者自己了。
	if (mb_tgtBlock == nullptr)
    {
		// print log: Warning: Mem has be deallocated.
		std::cout << "Warning: Mem \"" << _pAddr << "\" has be deallocated." << std::endl;
		this->tm_memMtx.ThreadUnlock();
		return;
	}
    // 接下来的内容都是内存释放以及链表操作，这里不再过多赘述。
	mb_tgtBlock->mb_previous->mb_next = mb_tgtBlock->mb_next;
	if (mb_tgtBlock->mb_next == nullptr)
	{
		free((void*)mb_tgtBlock);
		this->tm_memMtx.ThreadUnlock();
		return;
	}
	mb_tgtBlock->mb_next->mb_previous = mb_tgtBlock->mb_previous;
	free((void*)mb_tgtBlock);
	this->tm_memMtx.ThreadUnlock();
}
```

在这些工作做完后，我们的内存管理单元工作暂时告一段落，接下来我们要做的就是开始设计Release模式的内存管理单元了，这里的内容比较难以理解，但本人也会竭尽全力去用简单易懂的方式和各位说明，所以接下来建议各位可以稍微复盘一下前面的知识，同时可以冲杯咖啡休息一下，在所有准备工作完成后继续我们接下来的内容。

## 2. 内存管理（第三部分：Release模式，理论）

在上一篇博文里面我曾说过，单个内存空间的分配与释放过程要占用一部分的CPU时间，举个简单的例子说明一下：我们假设某家银行有很多保险柜，管理保险柜的管理员我们假设他就是CPU，如果我们此时要将我们的资料文件保存进去，那么首先，管理员需要通过一定的手段来找到适合存放我们资料文件的保险柜，有时还有可能是多个保险柜，即使是有查找表一类的比较快速的存在，但还是会消耗一部分时间，找到后，我们才能将我们的资料文件保存进去。同样的，我们要取出我们的文件时，管理员也要执行相似的活动再次消耗一部分时间。回到我们的引擎上来：如果我们需要引擎分配一个小的容量存储空间，它其实耗费的时间相比于大容量存储空间的分配过程没有多少变化，这显然很浪费。有没有什么解决办法？当然有，而且有很多。不过我们这里主要讨论的是来自虚幻引擎3的解决方法。（关于虚幻引擎4的源代码和文档我也没看，所以说虚幻引擎4究竟有没有继承使用这套内存管理方法也不好说）

在内存管理上，虚幻引擎3给出的解决方案是对于小于某一个值的内存，会通过内存池来分配，而大于这个值的内存，才会去使用直接分配策略。这么说有点抽象，让我来和各位细讲一下：

首先我们需要了解内存池的概念，内存池其实是一整块比较大的内存空间，而在它其中又分为了多个分配粒度，也就是一个个的小内存块，这些内存块在物理上是连续的，内存池的分配策略就是：在引擎初始化阶段会事先申请分配一至二个定长内存池，如果发现了小于分配粒度的内存分配请求，那么引擎会直接将内存池中空闲的内存块直接分配出来，这样会减少一部分的CPU时间。这时有同学会说：这还是没什么变化嘛，这也要CPU查找空闲内存啊。但是，请试想一下，是在一个有着几十甚至上百个进程以及复杂内存调度的操作系统环境下分配一块空闲内存方便还是在一个只有16个或者32个内存块的内存池里分配一块空闲内存空间方便？想必各位心里已经有了答案。在某一个内存池再无任何可分配空间后，引擎会将其归于“已占用”的内存池链表中，如果此时再有小内存需要分配操作，那么，引擎就需要再次像操作系统申请分配一另个内存池了，这样也确实会耗掉一部分时间，但比起每次都进行一系列复杂内存调度来说，这种内存池的分配方法降低了不少时间消耗。

接下来就是要讨论分配粒度的问题了，如何确定一个合适的分配粒度以及内存池也是一门学问，设立的太大那必然会造成内存空间的浪费，设立的太小却又会造成需要频繁向操作系统申请分配造成时间上的浪费，所以我们必须要找到一个折中方法。这里提供一个参考值：**页面大小**。我们可以以页面大小作为我们的分配粒度，然后以“页面大小 * 2的n次幂”作为内存池大小，因为操作系统每次分配内存必然是以页面大小为一个单位来进行内存分配的，这个页面大小具体是多少，取决于CPU，不过目前市面上大部分CPU都是以4096字节为一个页面，也就是4kB/页。看起来是不是很小，并不是，甚至比起我们引擎运行时的内存分配情况，这简直要大上太多。接下来就是内存池的大小，既然我们的引擎是64bit的，那么我们就把内存池大小也设置为64kB吧（这个纯属个人喜好，与64bit没有直接关系），而且64是2的6次方，也容易进行内存池的内存块的分割。

在处理完这些问题后，我们就需要开始进行相关内容的设计了，所以接下来请做好准备，一大波僵尸正在来袭（笑）。

## 3. 内存管理（第四部分：Release模式，实现）

让我们首先从内存池的实现开始讲起，其实在真正的虚幻引擎3中，内存池的实现是一共有42种的，这42种内存池在大小上并无区别，最主要的区别是在分配粒度上，甚至精细到8字节大小的内存分配，最多是32kBytes。因为没有人知道在具体的情况下具体需要分配多少内存，分配的要求视情况而定，这42种内存池的分配粒度取值其实也是经验数字，也就是通过穷举法尽量得到合适的分配粒度。不过鉴于本人目前过草的技术力，编写这么多的内存管理链的管理逻辑怕不是要疯掉，照着虚幻引擎的标准去制作引擎也没办法在明年毕业答辩前准备好，所以综合考虑，我目前会设计4种不同分配粒度的内存池，分别是2kBytes、4kBytes、8kBytes、16kBytes（换算过来就是半页长、全页长、双页长、四页长）。当然，这些都是以页面大小为4kBytes为基础。基于此所分配出来的内存块数量也不同，分别是32块、16块、8块、4块，其实还是可以再分下去的，但是已经没有必要了，再分下去的话只会造成内存分配过于频繁占据大量时间的结果，这显然是我们不想看到的。

在开始实现之前，让我们先放出这么几个宏定义：

```c++
#define SP_MEM_POOL_PAGE_NUM 16
#define SP_MEM_POOL_BLOCK_A_NUM 32
#define SP_MEM_POOL_BLOCK_B_NUM SP_MEM_POOL_PAGE_NUM
#define SP_MEM_POOL_BLOCK_C_NUM 8
#define SP_MEM_POOL_BLOCK_D_NUM 4
```

其中ABCD分别代表四种相同大小但不同规格的内存池，宏定义所代表的整型数值为各种内存池内内存块的个数。

首先，让我们对内存池类进行一个定义，定义如下：

```c++
class SPMemPool
{
public:
    // 构造函数，参数分别是指向真实内存区域的指针、
    // 内存池总体大小、以及内存池的内存块个数。
    // 这里主要是使用“大小/个数”来计算当前内存池的StorageType
    // MemLength就是自定义的一个unsigned long long的b
	SPMemPool(
		void* _pPoolMem,
		MemLength _iPoolMaxSize,
		MemLength _iPoolBlockSize
	);
    // 析构函数，目前来看没什么好说的
	~SPMemPool();
	// 获取内存块计数器数据，在后面我们会用到，所以将其开放出来。
	int GetMemCounter() { return this->i_memCounter; }
	// 在目标内存池中，找到还未被存储的内存块所在的整数索引。
    // 关于这个整数索引是什么，我稍后会讲。
	int GetFirstBlankBlock();
	// 获取本条内存池的存储类型，其实就是内存池中存储块的数量
	int GetStorageType() 
	{
		return this->bp_array[0]->i_storageType;
	}
	// 下面两个是自增和自减的符号重载，主要是对内存块计数器的自增与自减。
	void operator++();
	void operator--();
	// 通过对符号【】的重载，我们可以省掉一部分繁琐操作便可达到读取索引对应的内存地址。
	void* operator[](int _index) 
	{
		return (void*)((this->bp_array[_index]) + 1);
	}
	// 切换内存块存储标识符。
	void SwitchStorage(int _index, int _iStatus) 
	{
		this->bp_array[_index]->i_isStorage = _iStatus;
	}
	// 对指向下一个内存池的指针成员的Get与Set方法。
	SPMemPool* GetNextPool() { return this->mp_next; }
	void SetNextPool(SPMemPool* _mpNext) { this->mp_next = _mpNext; }

private:
    // 指向下一个内存池的next指针
	SPMemPool* mp_next;
    // 内存块索引数组
	MemBlockPoint** bp_array;
    // 指向自己所管理的真实内存区域地址
	void* p_poolMem;
    // 内存池大小
	MemLength i_poolMaxSize;
    // 内存池对应的内存块大小
	MemLength i_poolBlockSize;
    // 内存块计数器
	int i_memCounter;
};
```

在上面的定义中，有这么一个东西需要给大家解释一下：整数索引。在我们内存池的定义中，有一个私有数据成员“bp_array”，接下来放出它的定义：

```c++
struct MemBlockPoint
{
    // 指向单一内存块地址的指针
	void* p_blockPos = nullptr;
    // 判定是否被存储
	int i_isStorage = 0;
};
```

在这里使用整型值来确定对应的内存块的存储状态而不是布尔值，只是因为为了内存对齐好看一些（不过Windows还是会按照16字节存储，对哦，诶嘿）。

综上所述，我们目前的一个内存池结构大概可以这样去表示：

![pic2.png](https://s2.loli.net/2022/01/30/boSyw8aRY5hNP6V.png)

在内存池的表述完成后，我们开始设计内存管理器，由于在前面我已经讲过我们的内存管理器拥有四种不同规格的内存池可以分配，所以我们至少得在内存管理器的私有数据成员中添加八个内存池指针，即每种规格的内存池都需要两个指针（blank以及full）来管理，因为我们知道频繁进行申请与释放操作的内存单元有很大的几率会存在在blank所指向的内存池里，所以设置两个内存池指针会方便在释放过程中快速定位我们所需要释放的单元。在这些概念都已然明了后，我们便可以放出内存管理器的定义了：

```C++
class SPRlsMemManage : public SPMemManage
{
public:
    // 构造函数与析构函数，目前暂时没有什么可以在里面完善的内容
	SPRlsMemManage();
	~SPRlsMemManage();

    // 继承自内存管理器基类SPMemManage的两个方法：内存分配与释放
	void* memAllocate(MemLength _uiLength, bool _bIsArray);
	void memDeallocate(void* _pAddr, MemLength _uiLength);

private:
    // 私有化复制构造函数，毕竟我们的内存管理器是要掌握全局内存分配的
	SPRlsMemManage(SPRlsMemManage&);

    // 在内存管理器析构的同时，对残存在内存池指针上的所有内存池进行强制释放
    // 下面的强制释放内存块方法也是同理。
	void TerminatePools(SPMemPool*&);
	void TerminateMemBlocks();

    // 多线程互斥量对象
	SPThreadMtx tm_memMtx;

    // 八个内存池指针
	SPMemPool* mp_mBlock;	// Minimum (half) page.
	SPMemPool* mp_mFull;
	SPMemPool* mp_eBlock;	// Entire page.
	SPMemPool* mp_eFull;
	SPMemPool* mp_dBlock;	// Double pages.
	SPMemPool* mp_dFull;
	SPMemPool* mp_qBlock;	// Quad pages.
	SPMemPool* mp_qFull;

    // 当分配的内存大于最大内存块的大小时，内存管理器会为其直接分配内存
    // 这里的原理和调试模式下的内存分配器一致，不再过多赘述。
	SPRlsMemBlock* mb_memBlock;

    // 内存池的各个参数：内存块大小，内存池大小等
	MemLength ml_poolSize;
	MemLength ml_pageSize;
	MemLength ml_mBlockSize;
	MemLength ml_eBlockSize;
	MemLength ml_dBlockSize;
	MemLength ml_qBlockSize;
};
```

参考上面的定义，我们也便可以给出release模式下内存管理器的结构表示：

![pic3.png](https://s2.loli.net/2022/01/30/ZxaSOr25IhqBfpc.png)

接下来，让我们一步步实现其中的各个方法，首先是内存分配，我们知道的是在分配前，分配器需要知道需分配的大小。这一点与vc runtime中new的定义是一样的，毕竟只有知道大小，我们才可以为其制定分配策略，所以我们大致可以这么做：

```C++
void* memAllocate(MemLength _uiLength, bool _bIsArray)
{
    if(/*待分配大小小于半页长*/)
    {
        // 分配最小池
    }
    else if(/*待分配大小位于半页长与全页长之间*/)
    {
        // 分配全页面池
    }
    else if(/*待分配大小位于全页长与双页长之间*/)
    {
        // 分配双页面池
    }
    else if(/*待分配大小位于双页长与四页长之间*/)
    {
        // 分配四页面池
    }
    else
    {
        // 直接分配
    }
}
```

接下来就是各个内存池的分配逻辑了，由于除了内存池规格外其余都差不多，所以这里只以最小池的分配逻辑说明：首先我们需要考虑的是最普遍的情况，即在已知存在的内存池中分配内存，那么我们首先要做的就是在这个已知存在的内存池中寻找空内存块，那么我们的逻辑可以这么写：

```c++
if(this->mp_mBlock->mp_next)
{
	// Allocate pointer from pool which is exist.
	void* p_tgtPointer;
    // 这里调用了MemPool里面的GetFirstBlankBlock方法：寻找第一个未被存储的存储块
	p_tgtPointer = this->mp_mBlock->mp_next->GetFirstBlankBlock().p_blockPos;
    // 对找到的内存块的属性设置
    // 其实这里我们完全不用担心我们找不到空块，因为我们的内存池在每次分配完后都会进行自检
    // 及池子是否已满，若是满了，则自动挂至Full指针。
	this->mp_mBlock->mp_next->GetFirstBlankBlock().i_isStorage = 1;
	this->mp_mBlock->mp_next->i_blockCounter += 1;
    // 这里我们需要考虑的是假如我们此时分配的内存恰好是最后一个空块
    // 也就是说当它被分配出去后，池子恰好满了。
    // 我们这个时候就要将这个内存池挂至Full指针。
	if (this->mp_mBlock->mp_next->i_blockCounter == this->mp_mBlock->mp_next->qword_storageType)
	{
		// Pool is full after allocate.
		this->mp_mBlock->mp_next->i_isFull = 1;
        
        // 以下是基础指针操作，不做过多的赘述。
		SPMemPool* mp_tempPointer = this->mp_mBlock->mp_next;
		mp_tempPointer->mp_previous = this->mp_mFull;
		this->mp_mBlock->mp_next = mp_tempPointer->mp_next;
		if (mp_tempPointer->mp_next)
			this->mp_mBlock->mp_next->mp_previous = this->mp_mBlock;
		mp_tempPointer->mp_next = this->mp_mFull->mp_next;
		this->mp_mFull->mp_next = mp_tempPointer;
		if (mp_tempPointer->mp_next)
			mp_tempPointer->mp_next->mp_previous = mp_tempPointer;
	}
	this->tm_memMtx.ThreadUnlock();
	return p_tgtPointer;
}
```

接下来我们要考虑的是第二种情况，即上一次分配后我们正好没有待用的空池了，这一次分配我们需要重新向操作系统申请分配一个新池，那么逻辑如下：

```C++
else
{
	// Allocate new pool.
	void* p_tempPointer = nullptr;
	p_tempPointer = malloc(sizeof(SPMemPool) + this->ml_poolSize);
	if (!p_tempPointer)
	{
		this->tm_memMtx.ThreadUnlock();
		return nullptr;
	}
	SPMemPool* mp_tempPool = (SPMemPool*)p_tempPointer;
	mp_tempPool->SPMemPool::SPMemPool
	(
		nullptr, this->mp_mBlock, this->ml_poolSize,
		SP_MEM_POOL_BLOCK_A_NUM
	);
    // 关于这里的+1是由于我们已经将指针强制转换为了SPMemPool类型，
    // 所以+1正好就是跳过一个SPMem对象所占据的内存空间
    // 而不是跳过一个字节，还请注意。
	MemBlockPoint* bp_tempPointer = (MemBlockPoint*)(mp_tempPool + 1);
	for (int i = 0; i < SP_MEM_POOL_BLOCK_A_NUM; i++)
	{
		mp_tempPool->bp_array[i].i_isStorage = 0;
		mp_tempPool->bp_array[i].p_blockPos = 
			(void*)(bp_tempPointer + (i * (this->ml_poolSize / SP_MEM_POOL_BLOCK_A_NUM) / sizeof(MemBlockPoint)));
	}
	mp_tempPool->bp_array[0].i_isStorage = 1;
	mp_tempPool->i_blockCounter += 1;
	this->mp_mBlock->mp_next = mp_tempPool;
	this->tm_memMtx.ThreadUnlock();
	return mp_tempPool->bp_array[0].p_blockPos;
}
```

内存分配的逻辑解决后，接下来就是内存释放了，我们也需要待释放内存的大小，毕竟这样可以让我们的内存管理器更快的定位内存所在位置，所以它也会具有一个类似于Allocate的框架：

```c++
if(/*待释放大小小于半页长*/)
{
    // 在最小池里面找
}
else if(/*待释放大小位于半页长与全页长之间*/)
{
    // 在全页面池里找
}
else if(/*待释放大小位于全页长与双页长之间*/)
{
    // 在双页面池里找
}
else if(/*待释放大小位于双页长与四页长之间*/)
{
    // 在四页面池里找
}
else
{
    // 在SPRlsMemBlock链里面找
}
```

接下来同理，我也会只以半页长的释放为例进行逻辑讲解：

```C++
// Pool deallocate.
if (_uiLength <= this->ml_mBlockSize)
{
    /*
    * 首先为了能快速定位需要释放的内存地址的位置，我们会在未满池中去找
    * 这是由于小内存单元的释放次数与其自身大小大致可以说是成反比，
    * 所以首先我们假设我们需要释放的内存存在于未满池中。
    */
	if (this->mp_mBlock->mp_next)
	{
		SPMemPool* mp_tempPointer = nullptr;
		mp_tempPointer = this->mp_mBlock->mp_next;
		while (mp_tempPointer)
		{
			for (int i = 0; i < mp_tempPointer->qword_storageType; i++)
			{
                // 由于我们是将指针存储在一个定长数组内，所以我们可以直接对数组进行遍历查找。
				if ((_pAddr == mp_tempPointer->bp_array[i].p_blockPos) && (mp_tempPointer->bp_array[i].i_isStorage))
				{
					mp_tempPointer->bp_array[i].i_isStorage = 0;
					mp_tempPointer->i_blockCounter -= 1;
					if (!mp_tempPointer->i_blockCounter)
					{
                          // 若是此时释放掉内存后我们的池子正好变成了空池，那么就将其释放掉。
						mp_tempPointer->mp_previous->mp_next = mp_tempPointer->mp_next;
						if (mp_tempPointer->mp_next)
							mp_tempPointer->mp_next->mp_previous = mp_tempPointer->mp_previous;
						free((void*)mp_tempPointer);
					}
					_pAddr = nullptr;
					this->tm_memMtx.ThreadUnlock();
					return;
				}
			}
			mp_tempPointer = mp_tempPointer->mp_next;
		}
	}
	
    /*
    * 这里就是第二种假设：在未满池中未找到我们需要的地址，那么就在已满池中去寻找
    */
	if (this->mp_mFull->mp_next)
	{
		SPMemPool* mp_tempPointer = nullptr;
		mp_tempPointer = this->mp_mFull->mp_next;
		while (mp_tempPointer)
		{
			for (int i = 0; i < mp_tempPointer->qword_storageType; i++)
			{
				if ((_pAddr == mp_tempPointer->bp_array[i].p_blockPos) && (mp_tempPointer->bp_array[i].i_isStorage))
				{
                     // 找到以后可不是只进行释放就完事了
                     // 因为释放掉以后它肯定就是未满池了，这时候我们就要将其挂至未满池的指针了。
					mp_tempPointer->bp_array[i].i_isStorage = 0;
					mp_tempPointer->i_blockCounter -= 1;
					mp_tempPointer->mp_previous->mp_next = mp_tempPointer->mp_next;
					if (mp_tempPointer->mp_next)
						mp_tempPointer->mp_next->mp_previous = mp_tempPointer->mp_previous;
					mp_tempPointer->mp_next = this->mp_mBlock->mp_next;
					if (this->mp_mBlock->mp_next)
						this->mp_mBlock->mp_next->mp_previous = mp_tempPointer;
					this->mp_mBlock->mp_next = mp_tempPointer;
					mp_tempPointer->mp_previous = this->mp_mBlock;
							
					_pAddr = nullptr;
					this->tm_memMtx.ThreadUnlock();
					return;
				}
			}
			mp_tempPointer = mp_tempPointer->mp_next;
		}
	}
	// print log: Warning: Mem has been deallocated.
    // 若是实在未找到，那别无他法，只能输出一个警告信息告诉开发者未找到或者已被释放。
	string info = string("Mem has been deallocated.");
	WarningLog(SHADOW_ENGINE_LOG, info);
	this->tm_memMtx.ThreadUnlock();
	return;
}
```



## 4. 本篇结语

这次貌似是目前码的字数最多的一次，也算是拖更了这么久攒一次大的放出来，毕竟是一个比较庞大的系统，说的详细一点还是比较好的，在达到了复盘思路的同时也有利于让各位理解，不过本人生草的表达能力也说不出个什么花来，如果可以让各位理解那是最好，如果没有理解的话也欢迎提出问题，本人会逐一解答。由于这是我的技术博文，并不涉及到任何商业运营，所以可能不会像商业运营的UP那样快速更新以及回复。虽说目前研考也结束了，但之后的许多事情也把人压得抽不出太多时间，所以博文更新时间还是不定期的，这一点还请大家多多谅解，为下一次的内容做一下预告：目前我们只是完成了设计与实现，所以我们接下来就需要将其顺利移植到我们的引擎上并能让引擎正常工作，再一个，我们也会去实现引擎内的基本对象系统，好的，暂时就先说到这里，祝愿看到这篇博文的各位新年快乐！下次见！（溜~）

![知识共享许可协议](https://i.loli.net/2021/05/21/FDg2VLNJhyT7ZAE.png)

本作品采用[知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](http://creativecommons.org/licenses/by-nc-sa/4.0/)进行过许可
