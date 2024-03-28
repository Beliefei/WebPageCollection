# LLVM全时优化_llvm c++-CSDN博客
垃圾内容，勿看，浪费时间

引子
--

由于实验室项目原因，从本科开始接触Clang和[LLVM](https://so.csdn.net/so/search?q=LLVM&spm=1001.2101.3001.7020)开始到现在已经有2年时间了，期间都是针对Clang做一些边边角角的工作，没有潜下心来好好研读一下LLVM的paper。最近闲下来读了Chris Lattner那篇引用率高达2227的论文《[LLVM: A Compilation Framework for Lifelong Program Analysis & Transformation](http://ieeexplore.ieee.org/xpls/abs_all.jsp?arnumber=1281665&tag=1)》，这篇博客权当记录读paper时的所思所想。

### 介绍

刚了解Clang和LLVM的时候，并没有意识到LLVM的强大之处，只是简单着眼于Clang和GCC的对比。视野还局限在编译器前端词法、语法以及语义分析上，本科的《编译原理》课程讲到语义点到即止，大部分人写编译练手还只是写个Lexer、Parser生成出LLVM IR，然后交给LLVM后端处理。大部分人都编译器后端和代码优化还是比较陌生的。

LLVM不仅仅是一个Code Generator，简单的将IR翻译成机器码，它是一个支持全时优化的编译框架。虽说Lattner在他的博士毕业论文中写到“LLVM is much less ambitious and is more like an assembly language”，但按照现在LLVM发展情况来看，野心是很大的。如今LLVM应用到了很多方面， 很多语言有基于LLVM IR的实现， 比如说HHVM就拿LLVM作为后端。

在Lattner看来，LLVM就是一个全时分析优化的一个编译器框架。这个框架的核心就是LLVM IR，然后在LLVM IR上应用了很多激进的分析优化算法，并基于LLVM IR结合了很多新颖的编译优化概念。灵犀志趣在《[前瞻-全时优化和LLVM-1](https://web.archive.org/web/20140220054351/http://www.lingcc.com/2010/04/07/10721/)》提到 **“LLVM用上了目前编译所开发的所有技术:静态编译优化，JIT，虚拟机，解释器，运行时技术等等。也注定了LLVM要有一整套的工具链和环境来完成上述工作。”**

### LLVM全时优化

下图是2012年LLVM获得"ACM Software Systems Award 2012"奖项时的介绍图，这张图简明扼要的展示了LLVM的整体架构。

![](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwMTEyMTQ0MDI5NDY0?x-oss-process=image/format,png)

各种语言经过前端编译后生成LLVM IR，然后在link-time执行一些过程间的分析优化，这一块是LLVM很重要的一部分，过程间分析既要兼顾多种语言，同时又需要保留高层次的类型信息来执行过程间的优化。从前端编译到后端优化，到链接时以及运行时都有相关优化在里面。

编译优化其实在各个阶段都存在，只是LLVM将其打通了，什么别名分析，数据流分析，公共子表达式消除，循环优化，寄存器分配和链接时的相关优化，这些经典的后端分析都可以在其上实现。另外运行时通过收集profiling信息对LLVM Code优化并重新编译成native code，另外“[Automatic Pool Allocation](http://llvm.org/pubs/2005-05-21-PLDI-PoolAlloc.pdf)”这种黑科技竟然都可以实现，基于LLVM IR还可以实现多种GC算法，见[Garbage Collection with LLVM](http://llvm.org/docs/GarbageCollection.html#gcroot)(_**GC一直是我想要深入研究的技术，等看完LLVM关于GC的源码后再来写文章分析**_)。

文中将优化分为5个阶段：编译时优化，链接时优化，装载时优化，运行时优化，以及闲时优化。

**编译时优化**。包括一些经典的编译优化知识，在特定语言的编译前端将源代码编译成LLVM IR时，可以执行一些相关的优化，文中提到这些优化操作可以分为3部分，

（1）执行语言相关的优化。例如“optimizing closures in languages with higher-order function”（_**由于对函数式语言了解不深，暂时按住不表。等以后有时间再来研究**_）  
（2）将源代码翻译成LLVM Code，保留尽可能多的类型信息，例如结构体、指针或者列表等信息。  
（3）在单个模块内部可以调用LLVM针对全局变量或者过程间的优化

编译前端没有必要非得将编译结果构造成SSA形式，LLVM可以进行stack promotion操作，只要局部变量地址没有逃出当前函数作用域，就可以将栈上分配的变量分配在寄存器上，毕竟寄存器是没有显式地址的。

LLVM也可以将局部结构体对象或者列表映射到寄存器上，用于构造LLVM IR所要求的SSA形式。这一块我感觉应该是比较难的一块，编译器对structure或者说是memory layout的优化都是很难的一块（_**等以后研究以后再来补**_）

另外虚函数决议和尾递归优化也可以推迟到LLVM code阶段，有些虚函数调用决议是完全可以在编译期解决的，使用一个call(jmp)指令就可以了。尾递归优化比较通用，基本上所有编程语言都有这样的需求，尾递归优化可以减少对栈（即内存）的消耗，也避免了创建stack [frame](https://so.csdn.net/so/search?q=frame&spm=1001.2101.3001.7020)或者销毁stack frame的开销。

**链接时优化**。LLVM IR目标文件进行链接时，进行一些过程间或者跨文件的分析优化。link-time是首次能够见到程序全貌的阶段，在这个阶段可以做很多激进的优化，就像我们以前探讨过的问题，虚表或者typeinfo是否能够真正的优化删除（如果虚表或者typeinfo没有在程序中使用的话，仅限于完整程序，所以函数的定义可见）。LLVM的链接时优化如下图所示：

![](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwMTEzMTYwNTM3ODM0?x-oss-process=image/format,png)

LLVM在链接时所做的最激进的优化莫过于DSA和APA。在DSA分析中，借助于LLVM比较充足的type information，在指针分析的基础上，可以构造出整个内存对象的连接关系图。然后对这个图进行分析，得到内存对象的连接模式，将连接比较紧密的结构对象，例如树、链表等结构体分配在自定义的一个连续分配的内存池中。这样可以少维护很多内存块，并且大大提高空间locality，相应的提高cache命中率。APA（Automatic Pool Allocation）能够将堆上分配的链接形式的结构体，分配在连续的内存池中，这种做法是通过将内存分配函数替换为自定义池分配函数实现的，示意图如下所示：

![](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwMTEyMjIwMzEzMzgw?x-oss-process=image/format,png)

另外一些在链接阶段进行的分析还有，调用图构建，Mod/Ref分析，和一些过程间的分析，例如**函数inline**，**死全局变量删除**（dead global elimination），**死实参删除**（dead argument elimination），**死类型删除**（dead type elimination），**常量传播**，**列表边界检查消除**（array bounds check elimination），**简单结构体域重排**（structure field reordering），以及**Automatic Pool Allocation**。

在编译时会汇总每个函数摘要信息（procedure summary），附在LLVM IR中，在链接时就无需重新从源码中获取信息，直接使用函数摘要进行过程间分析即可。这种技术大大缩短了增量编译的时间。函数摘要一直是过程间分析的重点，因为这种技术在不过分影响精确性的前提下，大大提高静态分析的效率。我的本科毕设就是关于改写Clang以支持简单的基于函数摘要的静态分析，研究生毕设题目《基于函数摘要的过程间静态分析技术》。

这里简单提一下，**结构体域重排（structure field reordering）**，刚开始接触到这个概念的时候，其实我没有立刻意识到这个优化的重要性和难度。还是我在理解代码优化时，思考太局限，没能将体系结构体、操作系统和编译原理等知识结合在一起。另外思考问题的方式太幼稚，其实遇到结构体域重排这个问题时，要相应思考以下问题：

*   为什么需要结构体域重排
*   结构体域重排应该怎么做
*   结构体域重排会不会带来其他影响

另外结构体域重排也可以和hot概念相结合，如下代码所示：

```
`// s has hot member and cold member
struct s{
	char c;     // hot member
	int i;      // cold member
	double d;   // hot member
}；

// we can split s to two struct
struct hots{
	char c;
	double d;
};
struct colds{
	int i;
};` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15


```

将结构体根据hot的程度分拆成两个结构体，可以针对两个结构体进行不同的优化，例如将hot结构体的member提升到register中等（_**留待仔细研究结构体分拆优化**_）。

和结构体域重排优化概念相关的另外概念是**padding**， **alignment**， **point compression**。前两个概念和结构体域重排直接相关，示意代码如下所示：

```
`struct s1 {
    char a; // here padding 3 bytes
    int b;
    char c;
    char d;
    char e; // here padding 1 bytes
}` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7


```

这个结构体的sizeof是12bytes，但是如果将这个s1结构体子域重排的话，8bytes就足够了，如下图所示。

```
`struct s1 {
    char a;
    char c;
    char d;
    char e;
    int b;
}` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7


```

上面的结构体就是当前最紧凑的形式，虽然当前的优化很有诱惑，但是这个做法却又很多的问题。

首先就是，要做reordering的前提就是，必须能够确保当前结构只在当前TU中，如果当前结构体在另外的TU中也存在的话，那么就会存在结构体内存布局不相容的情况。所以这样的优化只能在链接时进行。

另外，就是要相应修改所有对struct member进行引用的操作，在C/C++这些类型不安全的语言中，类型转换非常普遍，有可能另外一种类型的使用其实就是在当前类型内存上进行的，如下代码所示。如何识别跟踪这些类型转换是一个非常困难的问题，LLVM IR提供了type information来帮助执行这些优化。

```
`struct S {
    char a;
    int b;
    char c;
};

struct S_head {
    char a;
};

struct S_ext {
    char a;
    int b;
    char c;
    int d;
    char e;
};

struct S s;
struct S_head *head = (struct S_head*)&s;
fn1(head);

struct S_ext ext;
struct S *sp = (struct S*)&ext;
fn2(sp);` 

![](https://csdnimg.cn/release/blogv2/dist/pc/img/newCodeMoreWhite.png)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24
*   25


```

另一个和**reordering the fields**相关的概念就是**point compression**，这两者的目的都是用于提高内存使用率，使内存显得更为紧凑。和**reordering the fields**类似的两个优化概念是**Structure peeling**，**structure splitting**，这些都属于**Data Layout Optimizations**。

_**以上关于结构体域重排的内容参考自**_  
《[Cache-Conscious Structure Definition](https://docs.google.com/viewer?url=http://users.ece.cmu.edu/~schen1/cs745/paper_pres.ppt)》  
《[Implementing Data Layout Optimizations in the LLVM Framework](http://llvm.org/devmtg/2014-10/Slides/Prashanth-DLO.pdf)》  
《[Automated field re-ordering in C structs to avoid padding](http://stackoverflow.com/questions/867471/automated-field-re-ordering-in-c-structs-to-avoid-padding)》  
《[Why can’t C compilers rearrange struct members to eliminate alignment padding?](http://stackoverflow.com/questions/9486364/why-cant-c-compilers-rearrange-struct-members-to-eliminate-alignment-padding)》  
《[Memory Layout Optimizations of Structure and Object](https://gcc.gnu.org/wiki/cauldron2015?action=AttachFile&do=view&target=Olga%20Golovanevsky_%20Memory%20Layout%20Optimizations%20of%20Structures%20and%20Objects.pdf)》

**LLVM代码生成器**。在寄存器分配之前，LLVM会一直保持SSA形式。LLVM代码生成结构如下图所示：

![](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwMTEzMTYzNDQ2Mjg1?x-oss-process=image/format,png)

图中描述了在代码生成时用到的一些技术，《[LLVM/GCC Integration Proposal](https://gcc.gnu.org/ml/gcc/2005-11/msg00888.html)》在这封邮件中，Lattner详细阐述了LLVM在代码生成方面的问题，从邮件中可以看到2005年的GCC开发人员在面对LLVM时还是比较轻视的心态：)（_**LLVM代码生成留待深入研究**_）。

**装载时优化**。装载时优化论文并没有明确提到（_**留待研究**_）

**运行时优化**。在大部分人开来运行时优化，仅仅和JIT、虚拟机和CPU乱序发射以及cache相关，因为运行时优化只在运行时执行，JIT可以结合虚拟机在程序解释执行时识别出hot区域，以便能够将这些代码编译成native code，CPU乱序可以依据当前指令的相关性来执行指令重排等操作，而cache也可以将频繁出现的内容缓存到cache中以便加快执行速度。可是LLVM和这些貌似都不在怎么搭边儿，确实LLVM的运行时优化和它们不怎么相同。LLVM运行时优化和闲时优化相互结合来实现相关优化，LLVM会在代码生成的时候插入一些低代价的指令来收集运行时的信息（profiling information），这些信息用于指导闲时优化重新生成一份更加高效的native code。这个过程也可以重复多次来达到较优的效果。一个重要的用处就是识别hot loop region和hot path，然后再对这些热区域进行特殊处理。LLVM针对此的具体实现还没有看，等研究透彻了，再用写文章详述。

有一篇文章是关于如何在LLVM基础上查找热区域的《[Implementation of Path Profiling in the Low-Level Virtual-Machine (LLVM) Compiler Infrastructure](http://llvm.org/pubs/2010-12-Preuss-PathProfiling.pdf)》。

和这个概念相似的就是代码插桩，代码插桩可以在源代码级别插桩，也可以在AST树上进行插桩，同样可以在机器码上插桩。我第一个参与的项目就是基于Clang做一个源代码级别的插桩工具，其中用到了**AST Rewriter**, **AST Macther**和**LibTooling**库。其实虚拟机中的行为和这个也很类似也是在运行时收集相关profile信息，然后指导后面的工作。

**闲时优化（离线重新优化）**。由于LLVM IR是永久保存的，存储在native code中，其实我第一次看到这个方法时，也是大跌眼镜，竟然将LLVM Bytecode直接存储在可执行ELF或者PE格式文件中。但是除了这个方法外，好像没有其他比较好的方法了（还是自己太孤陋寡闻了，看来还有很多知识需要补啊）。

LLVM IR分为几种形式，**Plain-text**, **Binary**, and **In-memory Representations**。其中的binary形式应该就是关于如何将LLVM IR存储到可执行文件中的（_**待我复习一下ELF格式再来仔细研究这个问题**_）

闲时优化的一种就是读取运行时提取出的profiling information，然后指导代码生成器生成一份儿更为高效的代码。

闲时优化的另一种形式，就是在用户端根据用户特定需求或者针对特定机器来提取相关profiling information，来执行特定的优化，由于这是离线（idle time）进行的，所以能够执行更为激进和expensive的优化策略，这是虚拟机所不能做到的。同样一份儿代码可以直接分发到不同架构机器上，经过一段时间的运行可以很好的适应或调整到最佳的native code。

* * *

### LLVM IR

LLVM IR是LLVM能够实现这些分析和优化的基石，LLVM IR的介绍见《[LLVM Language Reference Manual](http://llvm.org/docs/LangRef.html)》，这里面详细介绍了LLVM IR的方面面。这里我们简单的复述一下论文中关于LLVM IR的介绍，LLVM的中间表示是将LLVM和其他系统区别开来的关键因素。

LLVM IR能够保存高层次的程序信息，例如type info等，同时又足够底层以保证语言无关。下面从指令集，类型系统，内存模型，异常处理以及离线表示和in-memory表示来讲述LLVM IR。

**LLVM指令集**。从前我们论述过指令集的分类，有基于栈的，基于寄存器的和基于运算器的。而且基于寄存器的虚拟机和基于栈的虚拟机的对比是一个很热的话题，涉及到运算效率，性能以及指令size各个方面。静态分析工具KLEE就是在基于LLVM IR实现了一个虚拟机来专门进行静态分析。LLVM IR是基于寄存器的指令集，满足RISC架构以及load/store模式，也就是说只能通过将load和store指令来进行CPU和内存间的数据交换。

LLVM IR指令集拥有普通CPU一些关键的操作，但同时屏蔽掉了一些和机器相关的一些约束，例如物理寄存器（基于栈的虚拟机就是因为不依赖于具体机器的物理寄存器，而获得了良好的移植性），流水线和一些调用约定（例如通过哪个寄存器来存储返回值或者是使用哪个寄存器来存储C++中的this指针）。

LLVM提供了足够多的寄存器来存储基本类型值（bool, integer, floating point和pointer），这些寄存器是SSA形式（[静态单态赋值-static single assignment form](https://zh.wikipedia.org/wiki/%E9%9D%99%E6%80%81%E5%8D%95%E8%B5%8B%E5%80%BC%E5%BD%A2%E5%BC%8F)），这种形式的UD链（use-define chain, 赋值代表define, 使用变量代表use）非常明确，方便在其上做优化。整个LLVM指令集仅包含31条操作码（操作码少，用来表示操作码的位数就比较少，5位就够了），LLVM通过删除相同功能操作码或者复用统一操作码来较少操作码的种类。

LLVM中的内存地址没有使用SSA形式，因为内存地址有可能会存在别名或指针指向，这样就很难构造出来一个紧凑可靠的SSA表示。LLVM同时将CFG图显示的表示在LLVM IR中（_**仔细研究LLVM IR后再详述**_），一个function就是一组基本块的组合，一个基本块就是一组连续执行的指令并以中指指令结束（包括branch, return, unwind, 或者invoke等），当然中止指令指明了欲跳转的目的地址。

**语言无关的类型信息， Cast 和 GetElementPtr**。LLVM最基础的一个设计就是其语言无关的类型系统。每一个SSA寄存器或者显示的内存对象都有其对应的类型。这些类型和操作码一起表明这个操作的语义，这些类型信息让LLVM能够在低层次code的基础上进行一些高层次的分析与转换（例如黑科技DSA和APA分析，或者简单结构体域重排）。

LLVM IR包含了一些语言共有的基本类型（void, bool, signed/unsigned, double, floating等），并给他们一些预定义的大小，从8bytes到64bytes不等。这些保证了LLVM IR的移植性。同时LLVM又包含了四种（仅有四种）复杂类型，pointer，arrays， structures和functions。这四种类型足够表示现有的所有语言类型，例如C++中的继承体系就可以用**结构体嵌套结构体**实现。

```
`struct Base1{
	int b1;
}
struct Base2{
	int b2;
}
// 继承体系使用嵌套结构体实现，其实也就是C++中POD类型
// 非POD类型的一些约束或者特定结构例如虚表被其他形式所表示
struct Derived {
	Base1 b1;
	Base2 b2;
	int d;
}` 

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13


```

这四种类型对于程序分析与优化也是至关重要的，例如涉及到结构体的point-analyes，call graph construct和structure field reordering等。

由于LLVM是语言无关的，所以也应该能够支持弱类型语言，声明时的类型并不可靠。为了支持类型转换，LLVM提供了一个cast操作来实现一种类型到任意类型的转换，该操作对于C/C++这种类型不安全的语言提供了语义支持。另外，为了支持地址运算，LLVM提供了getelementptr指令，该指令多用于取结构体子元素时（例如 “.” 和 “\[\]”）。如下代码所示：

```
`// C语句
X[i].a = 1;

// LLVM指令
%p = getelementptr %xty* %X, long %i, ubyte 3;
store int 1, int* %p;` 

*   1
*   2
*   3
*   4
*   5
*   6


```

getelementptr指令在进行地址运算的同时，还保存了类型信息。例如X的类型信息为%xty，给已知类型定义别名是非常必要的，否则LLVM IR在涉及到复杂类型代码时生成的LLVM中间代码会非常复杂。地址运算对分析优化非常重要，程序中一般有大量的结构体，指针等等，有了地址运算，来分析各个内存对象之间的关系就会非常方便有效。GC，DSA和结构体优化都是基于point-analysis来做的，而地址运算就是point-analysis的基础。

**显示内存分配和统一内存模型**。LLVM提供特定类型的内存分配，可以使用malloc指令在堆上分配一个或多个同一类型的内存对象，free指令用来释放malloc分配的内存（和C语言中的内存分配类似）。另外提供了alloca指令用于在栈上分配内存对象（通常指局部变量，只是显示表示而已），用alloca来表示局部变量在栈帧上的分配，当然通过alloca分配的变量在函数结尾会自动释放的。

其实这样做是有好处，统一内存模型，所有能够取地址的对象（也就是左值）都必须显示分配。这就解释了为什么局部变量也要使用alloca来显示分配。没有隐式地手段来获取内存地址，这就简化了关于内存的分析。

**函数调用和异常处理**。对普通函数调用，LLVM提供了call指令来调用附带类型信息的函数指针。这种抽象屏蔽了机器相关的调用惯例。这篇文论里，lattner着重介绍了其异常处理机制，为了实现[异常机制](http://www.chasewoerner.org/exh.pdf)，LLVM提供了invoke和unwind指令。invoke指令指定在栈展开的过程中必须要执行的代码，例如栈展开的时候需要析构局部对象等。而unwind指令用于抛出异常并执行栈展开的操作。

栈展开的过程会被invoke指令停下来，执行catch块中的行为或者执行在跳出当前活动记录之前需的操作。执行完成后继续代码执行或者继续栈展开操作。

```
`// C++ exception handling example
{
	AClass Obj;   // Has a destructor
	func();       // Might throw, must execute destructor
}` 

*   1
*   2
*   3
*   4
*   5


```

上述代码展示了一种需要“cleanup code”的情形，"cleanup code"是通过C++编译前端产生出来的。func()有可能会抛出异常，C++必须确保Obj的析构函数能够正确调用。为了实现这个目标，invoke指令必须能够让栈展开的过程停下来，然后执行析构函数，执行完后继续随着unwind指令进行栈展开操作。

```
`// LLVM code for the C++ example. The handler code specified by invoke executes the destructor.
...
  ; Allocate stack space for obejct:
  %Obj = alloca %AClass, unit 1
  ; Construct object
  call void %AClass::AClass(%AClass* %Obj)
  ;call ''func()''
  invoke void %func() to label %OkLabel unwind to label %ExceptionLabel

OkLabel:
  ; ... execution continue ...
ExceptionLabel:
  ; If unwind occurs, excecution continues here. First destroy the Object:
  call void %AClass::~AClass(%AClass* %Obj)
  ; Next, continue unwinding:
  unwind` 

![](https://csdnimg.cn/release/blogv2/dist/pc/img/newCodeMoreWhite.png)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16


```

func()函数如果正常执行完，则跳转到OkLabel标签处继续执行，如果执行过程中发生了exception则跳转到ExceptionLabel处执行"cleanup code".

任何包含try语句的函数调用都会用invoke指令来实现。类似try/catch以及C++异常机制涉及到的RTTI等都会通过C++运行时库来实现，LLVM异常机制并不直接参与到机制中。

**Plain-text, Binary, and In-memory Representations**。LLVM IR是LLVM编译框架的核心，所以提供了3种表示形式为其服务，这三种表示可以无缝转换也不会有信息丢失。

* * *

### LLVM编译总架构

![](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwMTEzMTUyMzM1MDU2?x-oss-process=image/format,png)

LLVM的整体架构如上图所示。简单来说，编译前端将源码编译成LLVM IR格式的.o文件，然后使用LLVM Linker进行链接。Linker执行大量的链接时优化，特别是过程间优化。链接得到的LLVM code最终会被翻译成特定平台的机器码，另外LLVM支持JIT。本地代码生成器会在代码生成过程中插入一些轻量级的操作指令来收集运行时的一些信息，例如识别hot region。运行时收集到的信息可以用于离线优化，执行一些更为激进的profile-driven的优化策略，调整native code以适应特定的架构。

这种架构可以提供五个传统AOT编译所不具备的优势。Lattner认为这五点对于全时优化来说是至关重要的。

*   持续的程序信息
*   离线代码生成
*   用户主导的profiling及优化
*   透明的运行时模型
*   统一，全程序编译

这些优势很难同时获得，原有有二。

第一，离线代码生成(_#2_)通常是编译的最后一步，一般不允许在稍后阶段优化高层次的表示，只能针对native code进行优化。由于LLVM将字节码（LLVM IR）附在可执行文件中，所以可以保留高层次的信息以便后面阶段的再优化。

第二，整个生命周期的编译，传统来说只和基于字节码（Java, C#）的语言相关，然而他们不提供#4, 通常也不提供#2和#5。

当然LLVM也有自己的限制，首先语言相关的优化只能在编译前端实现，也就是生成LLVM code之前。LLVM不能直接表示语言相关的类型和特性，例如C++的类或者继承体系是用结构体模拟出来的，虚表是通过一个大的全局列表模拟的。另外需要复杂运行时系统的语言，例如Java，是否能够从LLVM中获益还是一个问题。在这篇文章中，Lattner提到，他们正在研究将Java或者CLI构建在LLVM上的可行性。

新想法的诞生从来都不是一夜之间出现的，一定是掌握了足够多的知识，在不同问题的比较和知识碰撞中获得灵感，然后像一个襁褓中的婴儿一样缓步前进的。当然现在LLVM还存在很多问题，特别是跟应用很多年的工业级的编译器在某些方面还有差距，但是差距正在逐步缩小，附一篇Open64开发人员对LLVM的看法《[Open64业内外人士对LLVM和Open64的观点](https://web.archive.org/web/20140218134757/http://www.lingcc.com/2010/06/15/10978/)》。

* * *

附：看这篇论文的时候，待研究的内容列表  
_**LLVM的GC源码**_  
_**函数式语言基于LLVM的实现**_  
_**编译器中"Memory Layout Optimizations Of Structures and Object"**_  
_**LLVM代码生成**_  
_**LLVM寄存器分配策略**_  
_**LLVM Linker**_  
_**LLVM IR**_