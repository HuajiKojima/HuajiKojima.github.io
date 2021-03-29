---
layout:     post                    # 使用的布局（不需要改）
title:      从新建文件夹开始构建UtopiaEngine（1）         # 标题 
subtitle:   要来力！！！          #副标题
date:       2021-03-29               # 时间
author:     赵永乐                      # 作者
header-img: img/Image/Article2.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - UtopiaEngine
    - 引擎开发日志
---

## 序言

在苦等了半年多之后，我终于开始了向往已久的实时NPR游戏引擎项目——**Utopia Engine**，这半年多一直为了构建这个引擎在做很多准备：多线程、动态链接库、脚本引擎、立即渲染GUI……统统吃了一遍（就差汇编没学了，话说这学期要开这门课来着，结果老师都已经翘课四周了(╯‵□′)╯︵┻━┻）。于是，等不及的我开始了Utopia Engine的构建项目（汇编的知识就一边做一边学吧）要赶在毕设前做完这个引擎工作量还是有些大的。当然，毕竟游戏引擎脱离不了计算机图形，所以，本博客里的计算机图形学相关内容也不会停止更新，敬请期待！

编写此文的目的也很简单：就是为了做一个记录。在以后的构建过程中防止出现错误而无法找到曾经埋下的隐患，并且同时也是学习相关技术的一个笔记，当然，我会尽量把这个系列写的通俗易懂，可以让看到这个博客同时也想要构建自己的游戏引擎的各位遵循着本系列做出自己的游戏引擎，是个有点偏向于教程的开发日志。但是并不是所谓的“0基础速成班”，至少各位是要有较为足够的C++语言开发经验以及计算机图形学的基础知识。如果两者都没有，那么各位在啃本人写的这些文章时恐怕就要有些费劲了。

## 1.项目设计

虽然说我对这个引擎渲染目标的定位是实时NPR（也就是实时非真实渲染），但其实效果更偏向于NPR里的“Toon Shading”，即卡通渲染。毕竟也是被新海诚导演制作的电影系列所惊艳到，所以希望通过实时渲染将这“虽然很假，但假的漂亮”的画面表现出来。不过目前由于本人的技术能力太过于生草以至于如果直接进行NPR的渲染实现不知道会做出什么屑作来，并且目前现存的成功NPR技术实例也并不是完全在引擎层面上去实现的，因为美术风格可是破坏渲染统一性的最大原因，著名的如万代南梦宫的《究极风暴》以及《罪恶装备X》几乎都是从贴图层面去实现真实的“假”细节（这可累坏了不少美工和TA），所以本人目前并不打算在Utopia Engine上实现实时NPR渲染，着重实现目前较为流行的PBR（基于物理渲染）技术，也就是真实渲染领域，要求不高，只要能通过本引擎做出一个可以跑的中小体量的3D游戏即可，这也就是目前本阶段的目标。接下来的事情就交给以后的我吧（拜托了，另一个我！）

## 2.初始化工作

### 2.1 项目构建

首先，声明一下，我这里的工作环境是微软的Visual Studio 2017 Community（以后均简称VS2017），使用的图形api如果不出意外的话应该就是OpenGL，WinSDK是 10.0.17763.0，不过，由于WinSDK所造成的编译失败或者是其他调试中的问题并不常见，所以这里可以忽略。

那么接下来开始构建项目，如何在VS2017里创建一个完整的C++空项目与解决方案并且为它创建本地与远程git代码库就不用我过多赘述了，相信看到这里的大家都明白。不过我们接下来并不会在项目里创建我们的第一个代码文件并且开始莽代码。因为VS2017自动创建的生成目录、中间目录、项目目录文件结构并不怎么符合我们的需求，只要记住项目名即可。

接下来要介绍一位熟悉的陌生人：premake工具，说它熟悉是因为想必各位都应该听说过CMake，这个premake和CMake的作用一样，都是用来做跨平台项目构建的工具。说它陌生是因为这个名字是真的陌生（至少我周围圈子的同学都没听过），使用这个工具的一个最主要原因是它不用在外网下载MinGW（￣へ￣），而且体量极小，一个Lua脚本就可以完成所有配置并且生成你想要的平台版本，缺点就是没有GUI，你得面对CMD无尽的黑暗（这里强力推荐微软的Windows Terminal，有了它你甚至可以将你的CMD装扮成初号机的显示屏）。不过操作很简单，而且官方超详细的Wiki也足以弥补这些缺陷。

如果你没有premake工具的话，请前往premake的[GitHub项目主页](https://github.com/premake/premake-core)下载对应操作系统的release压缩包。解压后你会得到一个premake5.exe可执行文件，将这个文件放入你的解决方案根目录里，接下来再在你的根目录里新建一个Lua脚本文件（文件后缀名是 **“.lua”**，文件名必须为premake5），在里面敲入如下Lua脚本代码：

```lua
workspace "UtopiaEngine"        -- 解决方案名称（填入你自己的引擎解决方案名称）
 architecture "x64"             -- 对应运行平台，如果你想兼容更老的32位系统，请再加x86选项

 configurations {               -- 配置类型：Debug，Release，Dist
  "Debug",
  "Release",
  "Dist"
 }

-- 全局变量：描述输出文件路径，因为无论是中间目录还是输出目录都有一部分相同的，所以将它们提取出来
-- 和VS2017一样，路径定义都可以用特殊格式转义符来表示，具体转义符表示意义请参阅premake工具的GitHub wiki，里面有详细解释，这里不过多说明
outputdir = "%{cfg.buildcfg}-%{cfg.system}-%{cfg.architecture}"

project "UtopiaEngine"          -- 引擎项目名,不要说你连项目和解决方案的区别都不知道哦
 location "UtopiaEngine"        -- 源文件所在目录名，这里建议和项目名保持一致
 kind "SharedLib"               -- 项目生成类型，这里选择SharedLib，在Windows平台上就是dll文件
 language "C++"                 -- 项目使用语言类型

 targetdir ("bin/" .. outputdir .. "/%{prj.name}")      -- 项目成品输出目录，Lua中使用“..”作为字符串与变量之间的连接符
 objdir ("bin-int/" .. outputdir .. "/%{prj.name}")     -- 项目中间目录

 files {                        -- 项目包含的文件类型，假如说你希望使用VS自带的.def文件代替__declspec(dllexport),那么请加上.def的声明
  "%{prj.name}/src/**.h",
  "%{prj.name}/src/**.cpp"
 }

 includedirs {                  -- 项目使用的第三方库的include目录，spdlog会在后面说明
  "%{prj.name}/vendor/spdlog/include"
 }

 filter "system:windows"        -- Lua里的条件判断语句，开始和中止边界以缩进为准，这里的意思是若目标系统为Windows则执行下列操作
  cppdialect "C++17"            -- 项目所遵循的语言标准
  staticruntime "On"            -- 静态运行时
  systemversion "10.0.17763.0"  -- 系统版本号，也就是WinSDK的版本号，如果不知道你自己的，可以填latest
  
  defines {                     -- 项目的全局宏定义，后面会解释两个宏定义的意义
   "UTOPIA_PLATFORM_WINDOWS",
   "UTOPIA_BUILD_DLL",
  }

  postbuildcommands {           -- 需要premake在生成项目时执行的命令
   ("{COPY} %{cfg.buildtarget.relpath} ../bin/" .. outputdir .. "/Sandbox")
  }

 filter "configurations:Debug"  -- debug配置相关设置，下同
  defines "UTOPIA_DEBUG"
  symbols "On"

 filter "configurations:Release"
  defines "UTOPIA_RELEASE"
  optimize "On"

 filter "configurations:Dist"
  defines "UTOPIA_DIST"
  optimize "On"


project "Sandbox"               -- 接下来是引擎编辑器的项目配置，名字什么都可以，大致与Utopia Engine项目配置一致
 location "Sandbox"
 kind "ConsoleApp"
 language "C++"

 targetdir ("bin/" .. outputdir .. "/%{prj.name}")
 objdir ("bin-int/" .. outputdir .. "/%{prj.name}")

 files {
  "%{prj.name}/src/**.h",
  "%{prj.name}/src/**.cpp"
 }

 includedirs {
  "UtopiaEngine/vendor/spdlog/include",
  "UtopiaEngine/src/UtopiaEngine"
 }

 links {
  "UtopiaEngine"
 }

 filter "system:windows"
  cppdialect "C++17"
  staticruntime "On"
  systemversion "latest"
  
  defines {
   "UTOPIA_PLATFORM_WINDOWS"
  }

 filter "configurations:Debug"
  defines "UTOPIA_DEBUG"
  symbols "On"

 filter "configurations:Release"
  defines "UTOPIA_RELEASE"
  optimize "On"

 filter "configurations:Dist"
  defines "UTOPIA_DIST"
  optimize "On"
```

接下来调用cmd命令行键入“premake5.exe vs2017”命令，其中如果你使用的IDE是VS2019，那么就填“vs2019”即可，回车后运行，运行结果如下：

![运行结果](https://ae01.alicdn.com/kf/H7b527e527ba44dfe8c5cea822a1c34f8L.jpg)

在继续之前先解释几个事情：

1. 关于为什么将引擎编辑器和引擎核心分离出来，这一点相信大家如果使用虚幻4引擎写过项目的话应该是深有体会的：自己项目的解决方案里通常在自己的项目之外也会包含一个名为UE4的项目（话说才发现Utopia Engine缩写竟然和虚幻一样），而这个项目就是引擎核心，是独立挂载于你的项目上的（ps：貌似UE4编辑器的代码就在核心里面，是要通过手动调用方法来略过生成编辑器的），再直白一点说就是你肯定不希望你的游戏里还塞着一个编辑器，占用存储空间不说，也会消耗一定的计算资源。所以分离开是很有必要的。

2. Lua脚本中的postbuildcommands选项的解释：
{COPY} %{cfg.buildtarget.relpath}：copy命令，如果是使用VS2017默认的生成目录，那么引擎核心dll文件是与编辑器可执行程序分开存储在两个不同的目录里，如果要是进行debug（尤其是针对引擎核心的debug），那么必须在每次debug之前先编译，然后手动将dll文件放入编辑器的生成目录里，麻烦。所以使用copy命令指示系统在每次生成完dll文件后会自动复制一份放在编辑器的生成目录里。

3. 目前我只写了关于Windows平台的命令脚本，为的是让项目的目录结构符合开发习惯并且减少无谓的工作量，所以这个premake脚本文件我并没有写MacOS以及Linux的生成命令，而且目前完全没有必要去在这两个平台上去调试代码（人穷，买不起Mac）。如果各位有需要，可以去premake的项目主页看看官方文档。

### 2.2 隐藏程序入口点

在引擎项目构建之前，我并没有想好究竟该为这个引擎配置一个怎样的脚本引擎。于是，我目前的想法是使用纯C++作为引擎的游戏开发脚本语言（与其说脚本倒不如说直接上源码，诶嘿！），就和UE4一样。不过，既然如此，就会很考验设计模式的基础了，即面向接口，因为游戏开发者可不想因为阅读你的源码而浪费大量的时间成本，而我们能做的就是尽量为他们提供通俗易懂并且功能实在的接口让他们去调用。

既然各位可以看到这里并且信得过我，那么我也相信各位已经有了一些图形API的调用基础（再不济你也应该拿Java的awt做过小游戏吧）。我们知道，一般图形程序无非就是由三个部分组成：即初始化、渲染循环以及释放资源。这三个部分我们可以将它封装在一个名为Application的类里（这里只是打个比方），然后用三个方法将这三个部分依次描述，再然后在main里面进行调用，这是目前最基础的做法。或许有人会说：“啊呀，你这不符合单一职责原则”。那你也可以封装成三个类，每个类只负责一个部分。但是游戏开发者（尤其是个人开发者）才不管你什么乱七八糟的“七大原则、二十多种模式”的条条框框，他们只想专注于游戏的实现，这时候如果让他们去自己实现main函数，恐怕就没人在再用你的引擎了（我就是这么过来的）。所以我们得隐藏程序入口点。

废话说了那么多，开始干正事。在你引擎的编辑器项目里新建一个cpp文件，在引擎核心里新建两个h文件，为了名称好记，暂且就叫做Engine.h(引擎头文件)、EntryPoint.h（入口点头文件）以及Application.cpp（编辑器源文件）。当然，名称只是打个比方，你可以按照你自己喜欢的名称来，但是你得记住相应的名称的文件作用是什么，毕竟我们还会在以后再用到它们，我们会在这三个文件中分别写入如下内容：

EnntryPoint.h

``` c++

int main(int argc, char** argv)
{
    // 这里是引擎初始化代码

    // 这里是引擎渲染循环代码

    // 这里是引擎运行结束前释放资源并且terminate的代码

    std::cout<<"Utah Teapot";
    return 0;

}

```

Engine.h

```c++

#include <iostream>
#include "EntryPoint.h"

```

Application.cpp

```c++

#include <Engine.h>

```

运行结果如下：（其实只要看第一行即可，因为下面是我后来加的log系统，如果是你的项目，应该会成功输出第一行的文字，log系统在之后的文章中我会讲到）

![运行结果](https://ae01.alicdn.com/kf/Had6c363e87594fe7946665a3a194faadz.png)

看起来貌似很简单，对没错，就这几行代码，我们完成了最重要的一步：入口隐藏。我们将入口隐藏在了引擎核心dll文件里，使得游戏开发人员不必将过多的精力放在main的实现上。这样看起来便有点样子了，不是么？

### 2.3 确定引擎内核规则

其实我也不知道该怎么称呼，总之就是想要将我们以后在引擎内核定义的对象、静态方法、函数等一系列的东西调用在游戏逻辑中的时候，可不止是一句简简单单的#include就可以解决问题，毕竟引擎内核是一串二进制代码构成的dll文件，即动态链接库。微软专门为动态链接库提供了两个语句：dllimport以及dllexport，调用方法如下：

```c++
// 假如说我在dll里的某个头文件里定义了这么一个函数，并且是对外提供调用的
__declspec(dllexport) void DllFunc()
{
    // Blablablablablablabla...
}

// 如果我要在另一个依赖此dll的项目里使用它，那么我必须在这个项目里声明
__declspec(dllexport) void DllFunc();

```

但是它太麻烦了，试想，如果每一个对外开放的函数都这么写，冗余的代码量增加不说，关键是在面对跨平台构建项目时，XCode可不认你的dllimport和dllexport，毕竟苹果的MacOS有着自己的一套SharedLib体系，这时候再去一个个改又会增加无谓的工作量。所以我们要通过某些手段来降低移植难度。

还记得在命令脚本里出现的两句全局宏定义吗：**UTOPIA_PLATFORM_WINDOWS** 以及 **UTOPIA_BUILD_DLL** 。接下来就是使用这些宏定义的时候了，我们可以为引擎核心创造一个专门用来进行SharedLib生成指令的头文件（文件名“Core.h”），代码如下：

```c++

#pragma once
#ifdef UTOPIA_PLATFORM_WINDOWS                          // 平台识别宏
    #ifdef UTOPIA_BUILD_DLL                             // 判断是否是Dll内定义
        #define ENGINE_API __declspec(dllexport)        // 如果是，则dllexport
    #else                                               
        #define ENGINE_API __declspec(dllimport)        // 否则，dllimport
    #endif                                              
#else
    #error engine is only build on windows at now!      //错误信息
#endif

```

先解释几个问题：

1. 我只是定义了适用于Windows平台的宏定义，其他平台大家可以根据编译器的要求来设定不同的分支。

2. 全局宏定义以及其他的宏定义的名称任君所爱，但是在定义完后请一定要记牢，以后还有用。

接下来让它被包含在Engine.h中。那么，在你的游戏项目中只要引入Engine.h并且在我们的以后书写中为类和函数定义前加上你所定义的 `ENGINE_API`，那么在游戏项目加载编译时，编译器会自动根据宏定义去判断究竟是使用import还是export而不用你在源文件中进行声明。

## 3.结束语

其实本引擎的结构在很大程度上与油管大佬Cherno的榛子引擎相像，因为我就是看着Cherno大佬的课程一步步来的，但是YouTube的听译字幕质量着实不咋样，导致我看的时候甚至一天只能看两个教学视频，B站也有好心的UP主搬运过来但由于是全英文的所以也劝退了不少人。我记得在虚幻引擎的官方Q&A里面有这样一句话：“代码是有版权的，但知识是无价的”。所以，如果说你的阅读能力不错，那可以跟着我一块来学习，我替你把该踩的雷先踩过，这样你也就轻松多了，对我来说也是在积攒开发知识的过程，何乐而不为？但是也不要抱太大希望哦，说不定哪天就断更了，诶嘿！o(￣▽￣)ブ

当然了，因为完成时间仓促以及本人技术力太过生草，肯定文中还有许许多多的问题，欢迎各位大佬们指正（如果只是想无脑喷的话，请出门左拐WB），以上。
