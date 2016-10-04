---
layout: page
title: C++强大背后
categories:
    - misc
---

在31年前(1979年)，一名刚获得博士学位的研究员，为了开发一个软件项目发明了一门新编程语言，该研究员名为[Bjarne Stroustrup](http://www.stroustrup.com/)，该门语言则命名为——C with classes，四年后改称为C++。


C++是一门通用编程语言，支持多种编程范式，包括过程式、面向对象（object-oriented programming, OOP）、泛型（generic programming, GP），后来为泛型而设计的模版，被[发现](http://www.erwin-unruh.de/meta.html)及[证明是图灵完备的](http://citeseerx.ist.psu.edu/viewdoc/download;jsessionid=AA022F3026015EF910CAEF5156901019?doi=10.1.1.14.3670&rep=rep1&type=pdf)，因此使C++亦可支持[模版元编程范式（template metaprogramming, TMP）](http://en.wikipedia.org/wiki/Template_metaprogramming)。C++继承了C的特色，既为高级语言，又含低级语言功能，可同时作为系统和应用编程语言。

C++广泛应用在不同领域，使用者[以数百万计](http://www.stroustrup.com/bs_faq.html#number-of-C++-users)。根据[近十年的调查](http://www.tiobe.com/index.php/content/paperinfo/tpci/index.html)，C++的流行程度约稳定排行第3位(于C/Java之后)。 C++经历长期的实践和演化，才成为今日的样貌。1998年，C++标准委员会排除万难，使C++成为ISO标准(俗称C++98)，当中含非常强大的[标准模版库（standard template library, STL）](http://www.sgi.com/tech/stl/)。之后委员会在2005年提交了有关标准库的[第一个技术报告（简称TR1）](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2005/n1836.pdf)，并为下一个标准[C++0x](http://en.wikipedia.org/wiki/C%2B%2B0x)而努力。可惜C++0x并不能在200x年完成，各界希望新标准能于2011年内出台。

流行的C++编译器中，微软Visual C++ 2010已实现[部分C++0x语法并加入TR1扩充库](http://msdn.microsoft.com/en-us/library/dd465215.aspx)，而gcc对[C++0x语法和库的支持](http://gcc.gnu.org/projects/cxx0x.html)比VC2010更多。

## 应否选择C++

### 哪些程序适宜使用C++?

C++并非万能丹，我按经验举出一些C++的适用时机。

*   C++适合构造程序中需求较稳定的部分，需求变化较大的部分可使用脚本语言；
*   程序须尽量发挥硬件的最高性能，且性能瓶颈在于CPU和内存；
*   程序须频繁地与操作系统或硬件沟通；
*   程序必须使用C++框架/库，如大部分游戏引擎（如Unreal／Source）及中间件（如Havok/FMOD），虽然有些C++库提供其他语言的绑定，但通常原生的API性能最好、最新；
*   项目中某个目标平台只提供C++编译器的支持。

按应用领域来说，C++适用于开发服务器软件、桌面应用、游戏、实时系统、高性能计算、嵌入式系统等。

### 使用C++还是C?

C++和C的设计哲学并不一样，两者取舍不同，所以不同的程序员和软件项目会有不同选择，难以一概而论。与C++相比，C具备编译速度快、容易学习、显式描述程序细节、较少更新标准（后两者也可同时视为缺点）等优点。在语言层面上，C++包含绝大部分C语言的功能（例外之一，C++没有C99的[变长数组VLA](http://en.wikipedia.org/wiki/Variable-length_array)），且提供OOP和GP的特性。但其实用C也可实现OOP思想，亦可利用宏去实现某程度的GP，只不过C++的语法能较简洁、自动地实现OOP/GP。C++的[RAII](http://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)（resource acquisition is initialization，资源获取就是初始化）特性比较独特，C/C#/Java没有相应功能。回顾历史，Stroustrup开发的早期C++编译器Cpre/[Cfront](http://en.wikipedia.org/wiki/Cfront)是把C++源代码翻译为C，再用C编译器编译的。由此可知，C++编写的程序，都能用等效的C程序代替，但C++在语言层面上提供了OOP/GP语法、更严格的类型检查系统、大量额外的语言特性(如异常、[RTTI](http://en.wikipedia.org/wiki/RTTI)等)，并且C++标准库也较丰富。有时候C++的语法可使程序更简洁，如运算符重载、隐式转换。但另一方面，C语言的API通常比C++简洁，能较容易供其他语言程序调用。因此，一些C++库会提供C的API封装，同时也可供C程序调用。相反，有时候也会把C的API封装成C++形式，以支持RAII和其他C++库整合等。

### 为何C++性能可优于其他语言?

相对运行于虚拟机语言（如C#/Java），C/C++直接以静态形式把源程序编译为目标平台的机器码。一般而言，C/C++程序在编译及链接时可进行的优化最丰富，启动时的速度最快，运行时的额外内存开销最少。而C/C++相对动态语言（如Python/Lua）也减少了运行时的动态类型检测。此外，C/C++的运行行为是确定的，且不会有额外行为（例如C#/Java必然会初始化变量），也不会有如垃圾收集（GC）而造成的不确定性延迟，而且C/C++的数据结构在内存中的布局也是确定的。有时C++的一些功能会使程序性能优于C，当中以内联和模版最为突出，这两项功能使C++标准库的`sort()`通常比C标准库的`qsort()`[快多倍](http://en.wikipedia.org/wiki/Sort_(C%2B%2B)#Comparison_to_qsort.28.29)（C可用宏或人手编码去解决此问题）。另一方面，C/C++能直接映射机器码，之间没有另一层中间语言，因此可以做底层优化，例如使用[内部（intrinsic）函数](http://en.wikipedia.org/wiki/Intrinsic_function)和嵌入汇编语言。然而，许多C++的性能优点并非免费午餐，代价包括较长的编译链接时间和较易出错，因而增加开发时间和成本，这点稍后补充。

我进行了一个简单全局渲染性能测试（512x512像素，每像素10000个采样），C++ 1小时36分、Java 3小时18分、Python约18天、Ruby约351天。评测方式和其他语言的结果详见[博文](http://www.cnblogs.com/miloyip/archive/2010/07/07/languages_brawl_GI.html)。

## C++常见问题

### C++源代码跨平台吗?

C++有不错的跨平台能力，但由于直接映射硬件，因性能优化的关系，跨平台能力不及Java及多数脚本语言。然而，实践跨平台的C++软件还是可行的，但须注意以下问题：

*   C++标准没有规定原始数据类型(如`int`)的大小，需要特定大小的类型时，可自订类型（如`int32_t`），同时对任何类型使用`sizeof()`而不假设其大小；
*   字节序（byte order）按CPU有所不同，特别要注意二进制输入输出、`reinterpret_cast`；
*   原始数据和结构类型的地址对齐有差异；
*   编译器提供的一些编译器或平台专用扩充指令；
*   避免作[应用二进制接口（application binary interface, ABI）](http://en.wikipedia.org/wiki/Application_binary_interface)的假设，例如调用函数时参数的取值顺序在C/C++中没定义，在C++中也不可随便假设RTTI/虚表等实现方式。

总括而言，跨平台C++软件可在头文件中用宏检测编译器和平台，再用宏、typedef、自定平台相关实现等方法去实践跨平台，C++标准不会提供这类帮助。

### C++程序容易崩溃?

和许多语言相比，C/C++提供不安全的功能以最优化性能，有可能造成崩溃。但要注意，很多运行时错误，如向空指针/引用解引用、数组越界、堆栈溢出等，其他语言也会报错或抛出异常，这些都是程序问题，而不是语言本身的问题。有些意见认为，出现这类运行时错误，应该尽量写入日志并立即崩溃，不该让程序继续运行，以免造成更大的影响(例如程序继续把内存中错误的数据覆写文件)。若要容错，可按业务把程序分割为多进程，像[Chrome](http://dev.chromium.org/developers/design-documents/multi-process-architecture)或使用fork()的形式。然而，C++有许多机制可以减少错误，例如以[string](http://en.wikipedia.org/wiki/String_(C%2B%2B))代替C字符串；以[`vector`](http://en.wikipedia.org/wiki/Vector_(C%2B%2B))或[`array`（TR1）](http://en.wikipedia.org/wiki/Array_(C%2B%2B))代替原始数组(有些实现可在调试模式检测越界)；使用智能指针也能减少一些原始指针的问题。另外，我最常遇到的Bug，就是没有初始化成员变量，有时会导致崩溃，而且调试版和发行版的行为可能不同。

### C++要手动做内存管理?

C++同时提供在堆栈上的自动局部变量，以及从自由存储（free store）分配的对象。对于后者，程序员需手动释放，或使用不同的容器和智能指针。 C++程序员经常进一步优化内存，自定义内存分配策略以提升效能，例如使用对象池、自定义的单向/双向堆栈区等。虽然C++0x还没加入GC功能，但也可以自行编写或使用现成库。此外，C/C++也可以直接使用操作系统提供的内存相关功能，例如内存映射文件、共享内存等。

### 使用C++常要重造轮子?

我曾参与的C++项目，都会重造不少标准库已提供的功能，此情况在其他语言中较少出现。我试图分析个中原因。首先，C++标准库相对很多语言来说是贫乏的，各开发者便会重复地制造自订库。从另一个角度看，C++标准库是用C++编写的(很多其他语言不用自身而是用C/C++去编写库)，在能力和性能上，自订库和标准库并无本质差别；另外，标准库为通用而设，对不同平台及多种使用需求作取舍，性能上有所影响，例如EA公司就曾发表自制的EASTL规格，描述游戏开发方面对STL的性能及功能需求的特点；此外，多个C++库一起使用，经常会因规范不同而引起冲突，又或功能重叠，所以项目可能须自行开发，或引入其他库的概念或实现(如[Boost](http://www.boost.org/)/[TR1](http://en.wikipedia.org/wiki/C%2B%2B_Technical_Report_1)/[Loki](http://loki-lib.sourceforge.net/))，改写以符合项目规范。

### C++编译速度很慢?

错，是非常慢。我认为C++可能是实用程序语言中编译速度最慢的。此问题涉及C++沿用C的编译链接方式，又加入了复杂的类/泛型声明和内联机制，使编译时间倍增。在C++对编译方法改革之前(如[module提案](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2006/n2073.pdf))，可使用以下技巧改善：第一，使用[pimpl手法](http://en.wikipedia.org/wiki/Opaque_pointer)，因性能损耗应用于调用次数不多的类；第二，仅包含必要头文件，并尽量使用及提供前置声明版本的头文件（如`iosfwd`）；第三采用基于接口的设计，但须注意虚函数调用成本；第四，采用[unity build](http://buffered.io/posts/the-magic-of-unity-builds/)，即把多个cpp文件结合在一个编译单元进行编译；第五，采用分布式生成系统如[IncrediBuild](http://www.xoreax.com/)。

### C++缺乏什么功能?

虽然C++已经非常复杂，但仍缺少很多常见功能。 C++0x作出了不少改善，例如语言方面加入Lambda函数、闭包、类型推导声明等，而库方面则加入正则表达式、采用哈希表的`unordered_set`／`unordered_map`、引用计数智能指针`shared_ptr`／`weak_ptr`等。但最值得留意的是C++0x引入多线程的语法和库功能，这是C++演进的一大步。然而，模组、GC、反射机制等功能虽有提案，却未加进C++0x。

## C++使用建议

### 为应用挑选特性集

我同意Stroustrup关于使用C++各种技术的回应：

>Just because you can do it, doesn't mean that you have to.
>
>你可以做，不意味着你必须这么做。

C++充满丰富的特性，但同时带来不同问题，例如过分复杂、编译及运行性能的损耗。一般可考虑是否使用多重继承、异常、RTTI，并调节使用模版及模版元编程的程度。使用过分复杂的设计和功能，可能会令部分团队成员更难理解和维护。

### 为团队建立编程规范

C++的编码自由度很高，容易编写风格迥异的代码，C++本身也没有定义一些标准规范。而且，C++的源文件物理构成，较许多语言复杂。因此，除了决定特性集，每个团队应建立一套编程规范，包括源文件格式(可使用文件模版)、花括号风格。

### 尽量使用C++风格而非C风格

由于C++有对C兼容的包袱，一些功能可以使用C风格实现，但最好使用C++提供的新功能。最基本的是尽量以具名常量、内联函数和泛型取代宏，只把宏用在条件式编译及特殊情况。旧式的C要求局部变量声明在作用域开端，C++则无此限制，应把变量声明尽量置于邻近其使用的地方，`for()`的循环变量声明可置于`for`的括号内。 C++中能加强类型安全的功能应尽量使用，例如避免“万能”指针`void *`，而使用个别或泛型类型；用`bool`而非`int`表示布尔值；选用4种C++ cast关键字代替简单的强制转换。

### 结合其他语言

如前文所述，C++并非适合所有应用情境，有时可以混合其他语言使用，包括用C++扩展其他语言，或在C++程序中嵌入脚本语言引擎。对于后者，除了使用各种脚本语言的专门API，还可使用[Boost](http://www.boost.org/doc/libs/1_42_0/libs/python/doc/index.html)或[SWIG](http://www.swig.org/)作整合。

## C++学习建议

C++缺点之一，是相对许多语言复杂，而且难学难精。许多人说学习C语言只需一本K&R[《C程序设计语言》](http://book.douban.com/subject/1139336/)即可，但C++书籍却是多不胜数。我是从C进入C++，皆是靠阅读自学。在此分享一点学习心得。个人认为，学习C++可分为4个层次：

*   第一层次，C++基础：挑选一本入门书籍，如[《C++ Primer》](http://book.douban.com/subject/4262575/)、[《C++大学教程》](http://book.douban.com/subject/2030264/)、或Stroustrup撰写的经典[《C++程序设计语言》](http://book.douban.com/subject/1099889/)或他一年半前的新作[《C++程序设计原理与实践》](http://book.douban.com/subject/4875599/)，而一般C++课程也止于此，另外[《C++ 标准程序库》](http://book.douban.com/subject/1110941/)及[《The C++ Standard Library Extensions》](http://book.douban.com/subject/1868179/)可供参考；
*   第二层次，正确高效地使用C++：此层次开始必须自修，阅读过《([More](http://book.douban.com/subject/1241385/))[Effective C++](http://book.douban.com/subject/1842426/)》、《([More](http://book.douban.com/subject/1244943/))[Exceptional C++](http://book.douban.com/subject/1967356/)》、[《Effective STL》](http://book.douban.com/subject/1792179/)及[《C++编程规范》](http://book.douban.com/subject/1480481/)等，才适宜踏入专业C++开发之路；
*   第三层次，深入了解C++：关于全局问题可读[《深入探索C++对象模型》](http://book.douban.com/subject/1091086/)、[《Imperfect C++》](http://book.douban.com/subject/1470838/)、[《C++沉思录》](http://book.douban.com/subject/2970056/)、[《STL源码剖析》](http://book.douban.com/subject/1110934/)，要挑战智商，可看关于模版及模版元编程的书籍如[《C++ Templates》](http://book.douban.com/subject/2378124/)、[《C++设计新思维》](http://book.douban.com/subject/1119904/)、[《C++模版元编程》](http://book.douban.com/subject/4136223/)；
*   第四层次，研究C++：阅读[《C++语言的设计和演化》](http://book.douban.com/subject/1096216/)、[《编程的本质》](http://book.douban.com/subject/4722718/)(含STL设计背后的数学根基)、C++标准文件[《ISO/IEC 14882:2003》](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3797.pdf)、[C++标准委员会](http://www.open-std.org/JTC1/SC22/WG21/)的提案书和报告书、关于C++的学术文献。

由于我主要是应用C++，大约只停留于第二、三个层次。然而，C++只是软件开发的一环而已，单凭语言并不能应付业务和工程上的问题。建议读者不要强求几年内“彻底学会C++的知识”，到达第二层左右便从工作实战中汲取经验，有兴趣才慢慢继续学习更高层次的知识。虽然学习C++有难度，但也是相当有趣且有满足感的。

数十年来，C++虽有起伏，但她依靠其使用者而不断得到顽强的生命力，相信在我退休之前都不会与她分离，也希望更进一步了解她，与她走进未来。

本文原于[《程序员》](http://www.programmer.com.cn/)2010年8月刊揭载。