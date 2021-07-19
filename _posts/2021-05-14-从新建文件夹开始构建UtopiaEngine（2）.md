---
ayout:     post                    # 使用的布局（不需要改）
title:      从新建文件夹开始构建UtopiaEngine（2）               # 标题 
subtitle:   说好的断更……才怪 #副标题
date:       2021-05-14              # 时间
author:     赵永乐                     # 作者
header-img: img/Image/Article2.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - UtopiaEngine
    - 引擎开发日志
---

## 本章序言

摸了两个月的鱼，又一次拾起了自己引擎的框架，开始完善引擎系统，如果非要用现实中的什么东西比喻的话，那么我们目前实现的框架连个脚手架都不是。把这项目这样晾着显然不符合本人的风格，而且要作为毕业设计的东西可不能蒙混过关。所以现在成了既要准备研究生考试又要忙于设计框架并编码的情况，生活已经充实到必须得抽空来写blog了。

还有一件事，就是我们的引擎现在的构建步骤可能要与我曾经参考的Cherno大佬的不同了，其中一个原因是因为他的game engine系列还在更新，对代码的修改也相比刚开始有很大区别，目前榛子引擎架构只有大体上与曾经视频中讲述的一致，具体代码实现有许多部分已经不同了。这也给在下徒增了不少麻烦。当然，本人的引擎在后期也会变成这样，可能你会在很长一段时间后才看到这个系列博文，而此时我发布在GitHub和码云上的引擎源码或许已经完全不同（本系列博文在连载时并不会放出源码，所以如果你看的时候本系列还未更新结束，或许不用太担心），这是不可避免的，不过文档的归档性至少比视频要好。还有一个原因就是本人的英语听力能力实在太过生草以及Cherno本人后期的语速实在太快，表示已经看不下去。如果对各位造成不便还请理解。

这段时间发现了一本比较好的书，是关于游戏引擎的，是由企鹅的程东哲大佬写的《游戏引擎原理与实践》（暂时忽略企鹅家那些游戏烂到家的口碑，那些都是策划的锅，至少企鹅的技术人员还是很强的），本引擎在后期的内存分配以及数据结构容器等部分可能会参考本书上的实现，各位也可以买来看看。

好了，正文开始，精彩继续。

## 1.应用程序接口

我们刚开始在引擎核心那里架设了入口点，但当我们在应用程序（游戏或编辑器）项目中写入任何处理流程时我们会发现引擎核心是并不会执行的。这很好解释，我们的引擎核心并不知道我们应用程序项目的存在，应用程序项目只是单向依赖引擎核心，并且更明显的原因是我们无法将应用程序项目中的处理步骤写入引擎核心的入口点的main函数里。强制性通过include来引入没人会知道发生什么事，恐怕只有编译器自己知道。

接下来就是解决方案，我们现在来创建一个应用程序接口，其实接口这个说法并不怎么严谨，按照严格OOP规则，接口内是不允许有方法实现的，但C++在这方面并不怎么“守规矩”以及我们的引擎核心有时也要实现其相关方法，但实在找不到个什么别的说法，所以就先勉强凑合一下。那么我们就先在引擎核心类内部声明并定义一个应用程序接口```BaseApplication```类，声明与定义如下：

```c++
// BaseApplication.h（声明）
#pragma once
#include "Core.h"
namespace Utopia
{
    // 还记得我在上一篇文章中说过的内核规则么？
    // 这里为了将我们这个应用程序接口暴露在dll外面，我们可以对类声明也这样做
    // 在类名前加上已经定义好的ENGINE_API即可，条件编译会保证调用正确，你可以用自己上次定义的宏
    class ENGINE_API BaseApplication
    {
    public:
        BaseApplication();
        virtual ~BaseApplication();

        void ExcuteLoop();
        virtual void ExcuteCallback();
    private:
    };
}

// BaseApplication.cpp（定义）
#include "BaseApplication.h"
#include <iostream>
namespace Utopia
{
    BaseApplication :: BaseApplication() {
        // 构造函数定义，用来在这里进行引擎核心相关的初始化步骤
        // 比如渲染框架的初始化，log系统的初始化等
        std::cout << "BaseApplication default constructor.\n";
    }
    BaseApplication :: ~BaseApplication() {
        // 析构函数的定义，用来释放已经被引擎核心调用的相关资源
        std::cout << "BaseApplication default destructor.\n";
    }
    void BaseApplication :: ExcuteLoop()
    {
        while(true)
        {
            // 把渲染以及每帧消息处理相关代码放在这里
            // 鉴于目前并没有开始渲染框架的构建，循环条件暂时以true代替，各位也可以随便编写一些条件测试一下
            // 但后续请记得删掉
            this->ExcuteCallback();
        }
    }
    void BaseApplication :: ExcuteCallback(){}
}
```



当然，老规矩，类名和命名空间名任君喜欢，但在后续调用中请记住它们的名字，以便调用。

这个时候呢，我们已经创建了引擎的应用程序接口类，接下来就是要在应用程序内创建应用程序接口类实现了，在我们的应用程序项目下新建一个.cpp文件即可，因为应用程序接口实现类是没有别的类会调用它的。声明与定义如下：

```c++
// Application.cpp（声明与定义）
#include "Engine.h"
#include <iostream>
class Application : public BaseApplication
{
public:
    Application();
    ~Application();
    
    void ExcuteCallback();
private:
};

Application :: Application()
{
    // 构造函数，用来初始化应用程序内的一些成员
    // 比如编辑器的UI框架，又或者是别的一些东西
    // 这里UI框架有些特殊，这里稍微剧透一下，本引擎打算使用的编辑器UI是著名的DearImGui
    // 但它的初始化过程必须在OpenGL相关API初始化并成功创建上下文之后，但这里不用担心，
    // 由于程序在运行时会首先运行接口类的初始化过程，完成后才运行本实现类的初始化过程。
    std::cout << "Application default constructor.\n";
}
Appication :: ~Application()
{
    // 析构函数，用来释放资源
    std::cout << "Application default destructior.\n";
}
void Application :: ExcuteCallback()
{
    // 用来将应用程序中需要在渲染与消息处理循环中处理的东西放在这里
    // 想必各位应该已经发现了这个函数其实是接口类BaseApplicaiton的一个虚函数，
    // 因为只有这样才可以让接口类运行应用程序中的处理流程（虚函数可真是个好东西）
    std::cout << "Application ExcuteCallback() has called\n";
}
```

细心的同学此时应该发现问题了，你的下一句便是：永乐，这里有点不对劲，即使已经声明了应用程序接口，但引擎核心还是不知道应用程序中实现类的存在，那么我们还是无法在入口点运行，如下：

```C++
// EntryPoint.h
int main(int argc, char** argv)
{
    BaseApplication* ba = new Application(); // 这里即使支持里氏替换原则，但编译器并不知道这个Application是谁
    ba->ExcuteLoop();
    delete ba;
    std :: cin.get();
    return 0;
}
```

这里不用着急，我们可以利用一个特性~~（Mojang：方块悬空不是bug，是特性！！！）~~：即声明与定义可以在不同的文件里面。我们可以在```BaseApplication```的声明文件里面添加这样一个函数的声明，也就是这样：

```C++
namespace Utopia
{
    class ENGINE_API BaseApplication{ ··· };
    // 我们在这里写上声明
    BaseApplication* ReturnAppInstance();
}
```

而我们会在```Application.cpp```里面这样去实现：

```C++
Utopia :: BaseApplication* Utopia :: ReturnAppInstance()
{
    return new Application();
}
```

这下我们就完成了一次“偷天换日”，我们将寻找实现的工作交给编译器，接下来要做的就是接一杯摩卡坐在躺椅上慢慢享受缓慢的MSVC编译过程……当然不是，距离成功运行我们还有些工作没做，那么接下来让我们一起来看看。

首先，就是```Engine.h```中的问题，我们虽然成功创建了应用程序接口，但我们并没有在```Engine.h```中包含应用程序接口的声明文件，以及我们并未包含引擎规则。所以我们会这样做：

```c++
#pragma once
#include "Engine.h"
#include "Core.h"
#include "BaseApplication.h"
```

以上就是目前```Engine.h```的完全体。

接下来是处理入口点中的一些问题：既然我们的入口点才是真正的执行体，那么我们便要定义如下执行体：

```C++
#include "Core.h"
#include "BaseApplication.h"
#include <iostream>
// 关于这里为什么要使用extern关键字：
// 编译器可没有IDE那么聪明直接进行跳转，由于编译器并未在同名.cpp文件内查找到相关函数声明
// 如果我们不做些什么的话，那么编译器就将错就错认为我们并未创建定义了，所以这时使用extern关键字
// 用来告诉编译器这个函数在别的地方已经定义过，让它扩大搜寻范围。
extern Utopia :: BaseApplication* Utopia :: ReturnAppInstance();
int main()
{
    BaseApplication* uBA = ReturnAppInstance();
    uBA->ExcuteLoop();
    delete uBA;
    std::cin.get();
    return 0;
}
```

这样便万无一失了，来按下f5键开始编译。最后运行结果应该是如下几句（前两句打印完后其实是会不再打印的，原因是我为循环设的条件为true，这时为了显示下面两句（运行析构，强制性关闭并不会运行析构），可以考虑加入某些循环成立条件）：

```	
BaseApplication default constructor.
Application default constructor.
Application default destructor.
BaseApplication default destructor.
```

不知大家发现没有，```BaseApplication```的构造和析构流程将```Application```的执行流程“包裹”起来。这样也便成功达到我们的目的：即先进行基础框架的初始化，再完成更高级模块的初始化，释放资源时正好相反。这样就能防止像Imgui初始化和释放资源时特殊情况了。

## 2. 日志系统

还记得我在上一篇文章说的日志系统么？这次就来填掉这个坑。这个部分是几乎所有应用程序都会有的一个子模块，比如CAD，模拟器（RPCS3，PPSSPP和PCX2等），以及你现在正在用的VS，各式各样的控制台程序等等……我们的引擎当然也不能少，至少在编辑器中我们是非常需要这个系统的，以及在游戏制作中的调试里我们也有很大的需要。所以，接下来开始构建日志系统，不过别担心，这个系统很简单，稍微一点点步骤就会完成。

### 2.1 spdlog

我们现在先在解决方案文件夹里新建一个文件夹Vendor（小摊贩？不过也差不多，后续我们引用的第三方库多起来的时候是不是就应该叫做Supermarket了？），专门在这个文件夹里放置各种第三方工具或代码。

我们的并不会自己从头去写一个日志系统，我们将采用一个第三方代码库：spdlog，这是一个调用非常简单，使用容易上手并且极其强大的专门的日志代码库，它默认有三种提示类型：error，warning，information，分别对应不同的提示颜色，你可以增加类型并自定义颜色，而且你甚至可以不仅让日志输出在控制台上，你也可以让它输出在任何你想要的界面上，不过鉴于本人技术力太过生草以及本引擎的体量，使用默认的设置就足以完成我们的需求。

前往GitHub去下载spdlog的源码（链接我就不放了，在GitHub搜索很容易就找到），记住，是下载源码，如果你的引擎项目添加了Git跟踪，你可以直接用```git module```命令扒取下来，这里不对这个命令做过多解释。下好源码后就可以将源码文件一股脑地全扔进Vendor文件夹里面。接下来请打开你的VS，我们要对我们引擎项目做些设置：

#### 2.1.1 新建项目（模块）

注意，这里的“项目”并不是指在引擎之外新建一个项目，而是VS解决方案中的“项目”，借此机会说明一下对应关系，其实我们的引擎项目对应的是VS中的解决方案，而VS中的项目的概念对应的是我们引擎项目中引擎模块的概念。正好就在这里进行严格规定，以后我会**将VS的解决方案称为解决方案或者引擎项目，VS的项目我们会称为引擎模块**，以此来避免概念混淆。

在本系列的第一篇文章发出后，有同学提出了反馈，说是新建项目用premake步骤还是比较麻烦，希望还是可以使用VS图形化界面来创建，本人想了一下觉得也是比较可行的，一个原因便是多次引擎项目重新载入花的时间太长，尤其是在后期引擎模块增多了以后那更是缓慢，而且使用脚本并不一定每次都会考虑周到将项目全部设置完毕，模块的依赖项太多时此缺点极其明显，类似于“热编译”这种的还是有些吃不消。所以接下来所有的项目构建过程本人都会采用VS自带的图形化界面创建，除了特殊之处需要说明外，其他步骤不放图。

首先在解决方案下新建一个新模块（VS选择“增加新建项目”），由于这个模块是专门为日志系统准备的，所以就起名叫做```EngineLog```即可，接下来在模块属性中添加附加目录，我们可以用VS提供的宏定义来编写附加目录项。如果此时我的spdlog的路径是：

```
D:/Project/UtopiaEngine/Vendor/spdlog
```

那么我们可以来这么写：

```
$(SolutionDir)Vendor\spdlog\include
```

这里```$(SolutionDir)```就是D:/Project/UtopiaEngine/路径的宏定义，这样就会在由于因为某些原因更改引擎项目目录的情况时不用担心得一条条更改依赖路径了。以下提供几个常用宏定义：

```
$(SolutionDir) // 解决方案路径
$(ProjectName) // 项目（模块）名称
$(Platform) // CPU平台名称，有x86，x64和arm三种
$(Configuration) // 项目属性，即Debug，Release，Dist等
```

接下来设置模块生成的二进制文件为“动态链接库（.dll）”，生成二进制文件的目录以及obj文件的目录和引擎核心与应用程序同步即可。（切记一定要将各个模块最终生成的二进制文件（.lib .dll .exe）均放在同一个文件夹内，premake5中的复制命令也可以完成，具体做法请参考上一篇）

#### 2.1.2 编写

在继续之前请为应用程序和引擎核心模块添加依赖项，即将我们的EngineLog作为它们的依赖项（即项目资源管理器中的依赖项以及模块属性中的附加包含目录均要添加），再然后为本模块新建一个文件夹src，代码文件均放在这里。完成此步骤之后，让我们开始编写相关代码。首先呢，我们需要和引擎核心一样规定内核规则，新建一个头文件```LogLibDefine.h```用来规定条件编译（当然不要忘记在模块属性的预处理器定义里面加上UTOPIA_LOG_DLLEXPORT哦）：

```c++
#pragma once
#ifdef UTOPIA_LOG_DLLEXPORT
	#define LOG_API _declspec(dllexport)
#else
	#define LOG_API _declspec(dllimport)
#endif
```

接下来就是创建相关类的声明与定义了：

```c++
// EngineLog.h
#pragma once

#include <string>
#include <memory>
#include <spdlog\spdlog.h>

#include "LogLibDefine.h"

// 设置两个宏定义来指定我要使用的日志输出类型，分为引擎日志和应用程序日志两部分
// 引擎日志主要用在编辑器以及其他的开发环境中，应用程序日志主要用在游戏程序调试或编辑器的相关信息中。
#define UTOPIA_ENGINE_LOG 1
#define UTOPIA_APP_LOG 2

namespace Utopia
{
    class LOG_API EngineLog
    {
    public:
        // 关于这里我为什么全部使用静态成员：
        // 由于日志系统的代码可以说几乎在引擎中的所有地方都会调用，如果使用非静态成员，那每次调用都要在相应类中
        // 设定一个日志类的成员对象，浪费了内存资源不说，可能还会造成不可必要的麻烦。
        // 其实关于这个还有一个更好的方法：将本模块转为静态库（.lib），这样便减少了模块调用之间的麻烦关系与限制。
        // 而且本模块并复杂，所以以静态库的形式在程序运行时就装载进内存对效率的影响影响不算大
        // 具体方法具体选择，大家可以尝试用静态库包装本模块。我目前在这里先使用动态库包装。
        static void LogInit();
        
        // 对参数解释一下：
        // 1. 类型是整型，用来存放我在上面的宏定义的，程序会根据宏定义的指定来选择日志输出方，即是引擎还是应用程序
        // 2. 类型是字符串，这很好懂啊，你想让输出什么信息，那就把它传进这个字符串里就好
        static void ErrorLog(int _iLogType, string _sLogInfo);
        static void WarningLog(int _iLogType, string _sLogInfo);
        static void InfoLog(int _iLogType, string _sLogInfo);
    private:
        // 关于这里我为什么使用智能指针：官方给的建议是这样，诶嘿
        // 但其实真实原因也是因为智能指针真的太香了，尤其是对于这种静态成员来说，我可以完全不用关心何时进行释放。
        static std::shared_ptr<spdlog::logger> s_CoreLogger;
        static std::shared_ptr<spdlog::logger> s_ClientLogger;
    };
}

// EngineLog.cpp
#include "EngineLog.h"
#include <spdlog\sinks\stdout_color_sinks.h>
namespace Utopia
{
    // 由于是静态成员，所以需要在这里实现一下
    std::shared_ptr<spdlog::logger> EngineLog::s_CoreLogger;
    std::shared_ptr<spdlog::logger> EngineLog::s_ClientLogger;
    
    // spdlog初始化步骤
    void EngineLog::LogInit()
	{
        // 这里是对Log的格式进行设置，最终输出结果是：
        // [xx:xx:xx]Utopia/APP：日志消息
        // 其他格式大家可以参考spdlog的官方文档自己去编写一个格式
		spdlog::set_pattern("%^[%T] %n: %v%$");
		s_CoreLogger = spdlog::stdout_color_mt("Utopia");
		s_CoreLogger->set_level(spdlog::level::trace);

		s_ClientLogger = spdlog::stdout_color_mt("APP");
		s_ClientLogger->set_level(spdlog::level::trace);
	}

	void EngineLog::ErrorLog(int _iLogType, string _sLogInfo)
	{
		string s_logErrInfo = "Cannot find log type, please check your code. Origin information: ";
		switch (_iLogType)
		{
		case UTOPIA_ENGINE_LOG:
			s_CoreLogger.get()->error(_sLogInfo);
			break;
		case UTOPIA_APP_LOG:
			s_ClientLogger.get()->error(_sLogInfo);
			break;
		default:
			s_CoreLogger.get()->warn(s_logErrInfo + _sLogInfo);
			break;
		}
	}

	void EngineLog::WarningLog(int _iLogType, string _sLogInfo)
	{
		string s_logErrInfo = "Cannot find log type, please check your code. Origin information: ";
		switch (_iLogType)
		{
		case UTOPIA_ENGINE_LOG:
			s_CoreLogger.get()->warn(_sLogInfo);
			break;
		case UTOPIA_APP_LOG:
			s_ClientLogger.get()->warn(_sLogInfo);
			break;
		default:
			s_CoreLogger.get()->warn(s_logErrInfo + _sLogInfo);
			break;
		}
	}

	void EngineLog::InfoLog(int _iLogType, string _sLogInfo)
	{
		string s_logErrInfo = "Cannot find log type, please check your code. Origin information: ";
		switch (_iLogType)
		{
		case UTOPIA_ENGINE_LOG:
			s_CoreLogger.get()->info(_sLogInfo);
			break;
		case UTOPIA_APP_LOG:
			s_ClientLogger.get()->info(_sLogInfo);
			break;
		default:
			s_CoreLogger.get()->warn(s_logErrInfo + _sLogInfo);
			break;
		}
	}
 }
```

完成了以上工作后，我们便可以开始下面的一步。

### 2.2 创建关联并部署进引擎

首先我们并不希望日志系统相关初始化步骤在每个调用它的模块里都执行一遍，那岂不是太麻烦了，~~濒危内存保护协会会提出抗议的~~，所以我们会让它在引擎核心老老实实地初始化后就不用再管其他的事情了。由于日志系统并不是状态机系统，所以也便不需要上下文的获取与释放，这样就让我们的行动更加灵活了。

老规矩，先为引擎核心创建相关模块依赖，两个依赖创建完成后，我们还要为引擎核心也包含spdlog的路径，在这些前置工作都做完后，我们便可以肆无忌惮地在引擎核心中调用其相关初始化方法，比如这样：

```C++
BaseApplication :: BaseApplication() {
    std::cout << "BaseApplication default constructor.\n";
    EngineLog :: LogInit();
}
```

当我们想要调用的时候就不需要再次初始化便可直接在想要调用其方法的函数体里调用。当然，别忘了为调用日志系统的模块创建依赖以及附加包含目录。运行效果的话大家可以参考上一篇那里的截图，那个就是我用了spdlog所创建的日志系统

## 3. 本篇结语

你看，多简单，就只有简简单单的两步，我们就创建了一个引擎的框架，其实目前看来这才算是一个应用程序框架，当然，距离游戏引擎框架还有一定的路要走，不过也不远了。再更上个三四回吧，我们大概就可以出搭建一个既具有底层渲染框架，事件系统以及音效系统的较为完善的游戏引擎框架。哦，做一个预告，下次更新我会开始搭建底层渲染框架以及部署我们引擎编辑器的UI底层。还请各位敬请期待。

![知识共享许可协议](https://i.loli.net/2021/05/21/FDg2VLNJhyT7ZAE.png)

本作品采用[知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](http://creativecommons.org/licenses/by-nc-sa/4.0/)进行过许可
