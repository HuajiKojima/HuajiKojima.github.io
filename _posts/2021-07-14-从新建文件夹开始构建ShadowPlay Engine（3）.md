---
ayout:     post                    # 使用的布局（不需要改）
title:      从新建文件夹开始构建ShadowPlay Engine（3）               # 标题 
subtitle:   从渲染窗口里说声HelloWorld！ #副标题
date:       2021-07-14              # 时间
author:     赵永乐                     # 作者
header-img: img/Image/Article2.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - UtopiaEngine
    - 引擎开发日志
---

## 本篇序言

各位可能看到博文的名字换了，也就是引擎名字换了，其实是在下想到了一个更棒的名字：皮影戏（ShadowPlay），取这个名字的含义是因为，游戏中的角色（Puppet）不也是由于我们的操作而动起来的么，电子游戏和皮影戏有很多相像的地方（可供人操作的纸偶，音乐，还有精彩的故事背景），不得不说古代人民的文化成果是最值得赞赏的（这或许也就是古时的小孩喜欢看皮影戏甚至自己想上去演的原因罢）。当然，决定改名后需要改动项目中的许多地方，不过不算太费劲，premake帮了很大的忙，这并不影响各位的阅读（如果你的引擎名字在之前已经是Utopia，那也不影响，因为架构都是一样的，除了命名外其余一律一致，要不然我也写不出博文啊（笑）），这次定名后就不再更改。还请各位多多包涵。

说了这么多，让我们开始今天的内容，在上一篇文章中我提出了接下来要为我们的引擎创建一个渲染底层，而且我在第一篇博文中也说到过，不出意外的话我们会使用OpenGL作为渲染用的API，看来目前为止的确没出什么意外（笑）。然后我也说过会创建引擎编辑器UI的底层，没错，今天我们会用到DearImGui，各位也可以将本文看做ImGui的入门教程，看起来今天的内容的确挺下饭的。好的，正文开始，精彩继续。

## 1. 渲染框架（第一部分）

目前我们只要让OpenGL帮我们绘制出个窗口就好，像Shader，VBO，VAO，Texture，Camera等这些相关部分我们会在后面逐一加入。鉴于目前我个人的学习安排与时间都很紧张，我准备从较为简单一些的2D系统入手开始进行开发（原因是曾经课设的时候自己一个人用Java做了个2D小游戏导致的自我膨胀，这个游戏在我的CSDN资源里面有，有兴趣的话可以看看），直到能做出一个中小型体量的2D游戏来，当2D系统完善后，再会在深入构建3D系统，~~谁赞成？谁反对？~~接下来让我们分几步走来完成本引擎的基础渲染框架。

### 1.1  新建渲染模块

我目前的决定是将渲染模块以动态链接库的形式独立出引擎核心，与日志系统是一样的。所以我们需要在解决方案下新建一个VS项目，名称为“```BasicStage```”（译作基础舞台，即使是皮影戏这种机动性较高的表演形式也需要有相关的戏台才能演出）。将其配置类型设置为“动态链接库（.dll）”，我们接下来开始向项目中添加附加包含目录，附加库目录以及静态库名，相信各位大部分也拥有OpenGL的配置经验，所以这里不做过多赘述，当然如果有这方面需要的话，可以参考LearnOpenGL或LearnOpenGL-CN，这里有详细的OpenGL配置过程。

当这些前置工作全部完成后，我们此时就可以开始构建我们的渲染模块。在本系列博文的第一篇就说明过：类似于这种实时渲染软件一般都是由三个部分组成的：初始化流程（Initialize），渲染和消息处理循环（Main Loop），终止流程（Terminate或Shutdown）。所以我们的渲染框架大致也按照这样去构建。

首先新建一个类，名为```RenderFrame```，这个类用来做渲染框架的主入口点，也就是说日后将所有渲染相关代码写在这个类里面，由```BaseApplication```类在主循环中调用相关渲染方法完成渲染过程。接下来就是编写该类的相关代码了，首先声明本类的方法：

```c++
class SHADOW_RENDER_API RenderFrame
{
public:
    // 关于这里我为什么将Application设置为友元类：
    // 因为目前引擎处在开发阶段，在调试某些部分时需要通过Application来进入引擎主入口点直接运行调试。
    // 在后面调试许多部分的时候，比如基础渲染单元，渲染链等就可能会跳过中间类直接需要进行调试，
    // 其实最大的原因是本人的技术力实在太草，如果各位有什么好想法也欢迎提出。
    // 不过后期当渲染系统开发完善后会去掉这里。
	friend class Application;
    // 构造函数，参数说明如下：
    // 1. 渲染窗口宽度：整型，默认值是1280；
    // 2. 渲染窗口高度：整型，默认值是720；
    // 3. 渲染窗口窗口标题：标准库字符串，默认值是“ShadowPlay ToyBox”；
    // 4. 应用程序是否包含编辑器：布尔，默认值是false
	RenderFrame(int _iScrWeight = 1280, int _iScrHeight = 720, std::string _sWindowTitle = "ShadowPlay ToyBox", bool 			_bWithEditor = false);
	~RenderFrame();
	
    // 用来在主循环中使用的渲染方法。
    void RenderCallBack();
    // 查询窗口上下文是否关闭的方法。
	bool WindowStatusQuery();
private:
	GLFWwindow* gl_windowCtx;
	std::string s_windowTitle;
	bool b_engineWithEditor, b_showUIDemoWindow;
	int i_scrWeight, i_scrHeight;
};
```

接下来我们来逐步实现本类中的方法。

### 1.2 配置OpenGL相关

首先让我们来实现本类的构造函数，首先我们需要知道OpenGL初始化相关过程，由于我们引入的OpenGL开发框架是GLFW以及GLAD，所以初始化流程大致也就是对以上两个框架的初始化，在LearnOpenGL中给出的初始化代码参考如下：

```c++
glfwInit();	// glfw初始化方法
glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 4);	// 设置OpenGL上下文主版本号（也就是OpenGL主版本号）
glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 5);	// 设置OpenGL上下文副版本号（也就是OpenGL副版本号）
glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);	// 设置OpenGL为核心模式（核心模式采用可编程渲染管线，也就是现代OpenGL）
// 创建窗口上下文（窗口宽度，窗口高度，标题）
GLFWwindow* window = glfwCreateWindow(1280, 720, “Hello!”, NULL, NULL);
if (window == NULL)
{
    // 窗口上下文判断
	glfwTerminate();
	throw std::exception("Failed to create GLFW window.");
}
// 将窗口上下文window设置为本线程的主要上下文
glfwMakeContextCurrent(window);
// glad初始化
if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
{
	glfwTerminate();
	throw std::exception("Failed to initialize glad");
}
glEnable(window);
```

接下来我们只要将上面部分的相关参数替换为我们类中的数据成员即可。哦，还有，别忘了在构造函数初始化列表里面初始化类成员。

在完成了构造函数后，我们可以开始渲染与消息处理循环调用的```RenderCallback```方法的实现，我们将以下代码抽离出主循环放入我们的方法：

```C++
// 在屏幕没有绘制任何内容时，将使用以下RGBA四个参数组成的颜色向量来刷新屏幕
glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
// 屏幕刷新相关函数
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

// 这里放置渲染代码

// 前后绘制缓冲区交换缓存函数
glfwSwapBuffers(window);
// 接受输入
glfwPollEvents();
```

最后是我们的析构函数，里面的内容也很是简单粗暴，一句```glfwTerminate()```即可。

这下，我们最最基础的渲染框架构建完毕，当然，此时并不能运行，就算运行了也没有窗口。在将OpenGL相关部署完成后我们要做的就是让它跑起来。

### 1.3 绘制窗口

其实各位中有绝大部分看到这里后会已经有些烦了，说：“永乐，你的这些过程在LearnOpenGL上都有啊，浪费了整整一大章就为说个这？”希望各位看官稍安勿躁，在下这么做原因有二：其一是由于本程序是将渲染流程和主入口点通过动态链接库完全隔离开，尽量去遵守“高内聚，低耦合”相关做法，而这方面在LearnOpenGL并没有提到。其实也就是为了满足各位在这方面的好奇心所做的踩雷行为（笑）。还有一个原因是看到这篇文章各位中的一部分对于OpenGL还是不太熟悉，本人也要尽量为这部分读者做好说明。

我在上一篇说过，我们引擎核心的```BaseApplication```类中的主循环缺少循环条件，我们一直粗暴地使用true作为循环条件肯定是不可行的。所以我们需要设置某些条件。我们能想到的是渲染主循环是与窗口息息相关的，当窗口关闭时，理所当然地程序应结束渲染主循环。以这个思路作为入口点我们就可以实现```WindowStatusQuery()```方法。glfw里面也提供了一个这方面的函数，即```glfwWindowShouldClose(GLFWwindow*)```，但我们总不可能在引擎核心中直接调用这个函数，我们需要的是只是个简单的布尔值，所以我们可以通过渲染框架中的```WindowStatusQuery()```方法来包装一下即可。这样就能降低平台依赖，以至于在以后将引擎移植到新的图形API时会更容易些。

这下我们的```RenderFrame```类就定义完成。接下来我们要做的就是让它开始工作。

在让引擎核心调用之前，请先为引擎核心创建相关依赖，至少得让引擎核心知道渲染框架的存在。在创建相关依赖后，我们就可以在引擎核心中调用相关的一大堆API了。首先我们应该在```BaseApplication()```里进行初始化，而不是```Application()```里。我们可以通过在```BaseApplication()```里设置参数来为渲染框架传递初始化参数。初始化结束后，我们需要将主循环中的循环条件从true改为我们的``WindowStatusQuery()```方法。接下来将渲染框架类中的渲染方法填入。最后就是析构函数，由于我为了编译速度使用前向声明导致我们只能使用相关指针初始化渲染框架，所以RAII机制在这里并不起作用，我们需要手动在析构函数中调用渲染框架的析构函数。

将以上步骤做完后，得到的```BaseApplication.cpp```完整代码如下（其中rf_Stage就是渲染框架的对象指针）：

```c++
BaseApplication::BaseApplication(int _iScrWeight, int _iScrHeight, std::string _sWindowTitle, bool _bIsEditor) :
	i_scrWeight(_iScrWeight), i_scrHeight(_iScrHeight), s_windowTitle(_sWindowTitle)
{
	EngineLog::LogInit();
	EngineLog::InfoLog(SHADOW_ENGINE_LOG, "BaseApplication default constructor.");
	this->rf_Stage = new RenderFrame(this->i_scrWeight, this->i_scrHeight, this->s_windowTitle, _bIsEditor);
}
void BaseApplication::ExcuteMain()
{
	while (rf_Stage->WindowStatusQuery())
	{
		this->ExcuteCallBack();
		this->rf_Stage->RenderCallBack();
	}
}
RenderFrame* BaseApplication::ReturnRFInstance()
{
	return this->rf_Stage;
}
void BaseApplication::ExcuteCallBack() {}
BaseApplication::~BaseApplication()
{
	EngineLog::InfoLog(SHADOW_ENGINE_LOG, "BaseApplication default destructor.");
	delete this->rf_Stage;
}
```

在一切都准备妥当后，我们可以f5一下查看运行结果，结果如下：

![pic1.png](https://i.loli.net/2021/07/19/syHBKbVM2odfGAS.png)

好了，这就是我们的引擎窗口，接下来的大部分内容都会通过这个黑漆漆的窗口展现，就像大阿尔卡纳塔罗牌中的愚者0一样，虽然什么都没有，但却有着无限的可能。接下来我们就要为编辑器做些事情了。

## 2. ImGui

其实各位将这部分看为ImGui（OpenGL版本）的中文教程也可以。而且在如今官方文档不怎么友好，中文社区不活跃甚至根本没有的情况下，在下自大地认为这是目前最适合入门的ImGui教程，当然，仅限使用OpenGL的ImGui版本。

ImGui其实是作者Occrnut对“Immediate Graphics User Interface（立即渲染模式图形用户接口）”的某种个性化缩写，后来更名为DearImGui（亲爱的立即渲染模式图形用户接口：我是育才中学高三年级的李华，衬衫的价格是9镑15便士……不好意思走错了），目前又改回了ImGui的名称。目前这个开源gui项目已经得到了包括但不限于英伟达、英特尔、暴雪以及Epic的官方赞助。所以还是有过硬的品质保证的。

由于本人技术力过草，本人并不可能为本引擎编辑器自己写一套立即渲染gui库，而现成就有这么好的一套工具，那当然白嫖啦~

目前这个gui项目在GitHub开源，各位可下载到它的源码包，但官方的readme中并未说明如何配置，只说了如何用，在下在经历了官方文档忽悠、帖子被灌水、阅读实例程序源码后总结出了本套gui库的基本用法。且听在下一一道来。

### 2.1 配置

首先我们在GitHub上搜索ImGui，找到后下载它的源码包，解压完成请在源码文件夹里提取（复制，不建议剪切）以下文件：

![pic2.png](https://i.loli.net/2021/07/19/ODwXc78uCm1ZRoI.png)

看着貌似源代码包里代码文件数量多得吓人，但实则真正起作用的就是这么几个文件。接下来为这些文件找一个新家：在引擎项目里新建一个VS项目：ImGuiStaticLib，将这个项目的配置方式设置为“静态库（.lib）”将这些提取出的文件放入项目中，当然别忘了要将ImGui的开源协议也放进去，在项目资源管理器里刷新一下解决方案目录，并将此项目中所有文件导入ImGuiStaticLib即可。

这次我将ImGuiStaticLib模块设置为lib静态链接库，其最大的一个原因是引擎中的许多地方都要去调用这套框架，如果不这样做的话，那每次都要为相关调用的模块里包含一大堆的.cpp文件，很是麻烦。生成lib文件后我们其实可以创建一个测试这套框架的驱动，在以后若要对ImGui进行更新，可以先用这套测试驱动测试运行效果。不过这里不做过多赘述，不用着急，看完本篇文章你也就自己会写了。

由于我们需要在引擎渲染框架以及我们的编辑器项目ToyBox（就是引擎名字为Utopia时的编辑器项目Sandbox）中使用ImGui，那么我们就要为这两个项目创建依赖。即添加附加包含目录、添加附加库目录以及添加输入项，这里附加库目录有点特殊，因为我们这次使用的lib是我们自己生成的，所以我们此次添加的附加库目录是我们二进制文件生成的目录，其他步骤和我们添加glfw时是一致的。

在为这两个项目都配置完毕后，我们就可以开始在项目中使用ImGui了。

### 2.2 部署

在我们将项目依赖配置好以后，就可以开始搭建框架，首先我们要明白的一件事情是，使用游戏引擎开发游戏的游戏开发者有很多是对底层API一无所知的，我们绝对不能将底层API暴露给游戏开发者让他们自己去调用，毕竟游戏开发者光是想gameplay的写法就已经够头秃了，如果我们将这些API暴露给他们让他们自己去做这些工作，那还不如自己用OpenGL写个游戏，为什么还要用引擎呢？所以我们至少要将ImGui的初始化，渲染设置，终止流程，上下文等概念让渲染框架去做，而不是甩给游戏开发者。

所以让我们再次打开我们已经写好的渲染框架类```RenderFrame```，让我们再次完善其中的实现方法：

#### 2.2.1 初始化

首先，ImGui也是需要初始化步骤的，所以我们让渲染框架类的构造函数去帮我们完成这个步骤。在渲染框架类构造函数内写入如下代码：

```c++
// 在写入这段代码前，请先在代码文件中包含“imgui_impl_glfw.h”以及“imgui_impl_opengl3.h”两个头文件
if (this->b_engineWithEditor)
{
    // 检查版本
	IMGUI_CHECKVERSION();
    // 创建ImGui的上下文
	ImGui::CreateContext();
    // 确定ImGui的输入输出对象，其实这里还可以设置字体，比如consolas等等，
    // 方法如下：
    // io.Fonts->AddFontFromFileTTF("font files.ttf", float fontsize, NULL, io.Fonts->GetGlyphRangesChineseFull());
	// 在不使用这个方法前，你是不能使用汉字在ImGui进行输出的，因为ImGui自带的字体不支持非拉丁语系
    // 如果非不信邪，那就做好面对乱码天书的准备吧，为了你好，真的
    // 所以你需要去自行下载支持汉字及其它非拉丁语系语言的字体并将其与渲染框架放在同一个VS项目目录里
    // 别忘了，你下载的字体同时也要复制一份放在二进制文件目录里
	ImGuiIO& io = ImGui::GetIO();
	(void)io;
    // 设置ImGui的颜色主题，有三种：
    // StyleColorsDark()		暗色主题
    // StyleColorsLight()		浅色主题
    // StyleColorsClassic()		经典主题
    // 大家可以每个都试一下看看视觉效果
	ImGui::StyleColorsDark();
    // 由于我们使用glfw库初始化OpenGL，所以这里要创建ImGui与glfw之间上下文的联系
	ImGui_ImplGlfw_InitForOpenGL(gl_windowCtx, true);
    // 创建ImGui上下文基于的OpenGL版本，应与glfw初始化时设置的一致，都是4.5版本，
    // 其实3.3版本（从3.3版本后为支持核心模式的现代OpenGL）及以后都可填，最大不得超过你的显卡驱动支持的版本
    // 关于显卡支持的OpenGL版本，大家可以下载gpu-z查看
	ImGui_ImplOpenGL3_Init("#version 450");
}
```

在这里我们使用了我们之前设置的布尔值```b_engineWithEditor```来让引擎确定是否初始化ImGui，因为这类实时渲染软件对性能的要求都比普通的软件系统要高，在应用程序不需要使用编辑器时，就没必要再对ImGui初始化了。

#### 2.2.2 终止

我们会将这个步骤放在渲染框架的析构函数中，需要填入的代码如下：

```c++
// 同理，使用类中内置布尔成员来确定是否执行
if (this->b_engineWithEditor)
{
    // 将基于OpenGL的依赖撤销
	ImGui_ImplOpenGL3_Shutdown();
    // 将基于glfw的依赖撤销
	ImGui_ImplGlfw_Shutdown();
    // 摧毁ImGUI的上下文
	ImGui::DestroyContext();
}
```

#### 2.2.3 渲染循环

我们将这个步骤放在渲染框架里的渲染函数中去执行，代码如下：

```c++
if (this->b_engineWithEditor)
{
    // 创建新的ImGui渲染帧
	ImGui_ImplOpenGL3_NewFrame();
	ImGui_ImplGlfw_NewFrame();
	ImGui::NewFrame();

	// 必须将所有ImGui的方法代码放在这里（比如窗口实体，控件等）
	
    // 对所有imgui命令进行渲染
	ImGui::Render();
    // 使用OpenGL对ImGui中的所有待绘制内容进行渲染
	ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());
}
```

这里使用布尔成员不只是为了防止不必要的步骤，也是为了防止应用程序里不小心在渲染对象中写入ImGui相关的方法从而因缺少必要上下文导致整个程序崩溃的情况。

### 2.3 创建第一个ImGui窗口

接下来我们就可以在渲染循环中创建一个ImGui窗体来检验我们的框架搭建是否正确，在2.2.3渲染循环中写入如下代码：

```c++
if(b_firstWin)
{
    // 开始绘制，参数分别是窗口标题以及确定窗口关闭的布尔变量的引用
	ImGui::Begin("DemoWindow", &b_firstWin);
    // 在窗体上输出文字
	ImGui::Text("Text Testing");
    // 开启主菜单条
	ImGui::BeginMainMenuBar();
	ImGui::EndMainMenuBar();
    // 颜色设置组件，v是一个长度为4的单浮点数数组，颜色数据全部存储在这里
	ImGui::ColorEdit4("Color,", v);
    // 颜色属性框，鼠标放在上面会显示颜色的详细信息，名称自定不唯一
	ImGui::ColorButton("Color", ImVec4(0.0f, 1.0f, 0.0f, 1.0f));
    // 结束绘制
	ImGui::End();
}
```

此时我们按下f5键生成并运行，我们会得到如下结果：

![pic3.png](https://i.loli.net/2021/07/19/Tbz64nXUGK7pScs.png)

![pic4.png](https://i.loli.net/2021/07/19/v3hdupJikGzQgcA.png)

成功了！我们成功地将ImGui部署在我们引擎的渲染框架上，离着成熟的游戏引擎又近了一步。

## 3. 上下文抽象

在经历了创建ImGui窗体成功的喜悦之后，请各位先冷静一下，我们现在要开始总结一些事情，一些很重要的事情。

在开发到这里的时候我们会发现，无论是ImGui还是更加底层的OpenGL，都是典型的状态机系统，驱动它们工作的便是上下文。而上下文这个概念是完全平台依赖的，也就是说OpenGL的上下文是绝对不可能在DX11（DX12、VULKAN、Metal以及PlayStation 5所用的图形API采用的不是状态机而是另外的概念模型，本处并不适用）设备上下文里面去用的，我们要降低引擎的平台依赖度，所以这也就意味着我们必须要将上下文抽象出来，所以我们创建上下文抽象类如下：

```C++
// 声明
class SHADOW_RENDER_API RenderContext
{
public:
	RenderContext() : gl_windowCtx(nullptr), imgui_renderCtx(nullptr) {}
	~RenderContext();

    GLFWwindow* ReturnGLCtx();
	ImGuiContext* ReturnImGuiCtx();
	void SetGLCtx(GLFWwindow*);
	void SetImGuiCtx();
private:
	GLFWwindow* gl_windowCtx;
	ImGuiContext* imgui_renderCtx;
};

// 定义
RenderContext::~RenderContext()
{
	glfwDestroyWindow(this->gl_windowCtx);
	this->imgui_renderCtx = nullptr;
	this->gl_windowCtx = nullptr;
}
GLFWwindow* RenderContext::ReturnGLCtx()
{
	return this->gl_windowCtx;
}
ImGuiContext* RenderContext::ReturnImGuiCtx()
{
	return this->imgui_renderCtx;
}
void RenderContext::SetGLCtx(GLFWwindow* _glWindowCtx)
{
	this->gl_windowCtx = _glWindowCtx;
}
void RenderContext::SetImGuiCtx()
{
	this->imgui_renderCtx = ImGui::GetCurrentContext();
}
```

相信各位之中有很多使用Java开发过bean，这个抽象出的上下文和JavaBean差不多（其实就是一堆数据成员和这堆数据成员的get与set方法），所以就不再过多赘述。

## 本篇结语

我们创建了引擎系统的基础渲染框架并为引擎配置了ImGui，并成功地创建了第一个编辑器窗口，这也说明我们的引擎基础渲染框架搭建成功，但这并没有完，其实在引擎渲染框架的许多地方还是没能摆脱较高的平台依赖性，而且我们还并未用引擎实现图形学中的HelloWorld——三角形，综上所述问题还很多，所以在我们的旅程还远没有结束，为下一次的文章做一个预告：下一次我们将会在渲染框架中实现基础渲染对象，渲染链，场景链这三样游戏渲染中必不可少的框架。好的，下次见~

![知识共享许可协议](https://i.loli.net/2021/05/21/FDg2VLNJhyT7ZAE.png)

本作品采用[知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](http://creativecommons.org/licenses/by-nc-sa/4.0/)进行过许可