# 失传的C结构体打包技艺 #

作者：Eric S. Raymond

原文链接：http://www.catb.org/esr/structure-packing/

## 谁应阅读本文 ##

本文探讨如何通过手工重新打包C结构体声明，来减小内存空间占用。你需要掌握基本的C语言知识，以理解本文所讲述的内容。

如果你在内存容量受限的嵌入式系统中写程序，或者编写操作系统内核代码，就有必要了解这项技术。如果数据集巨大，应用时常逼近内存极限，这项技术会有所帮助。倘若你非常非常关心如何最大限度地减少处理器缓存段（cache-line）未命中情况的发生，这项技术也有所裨益。

最后，理解这项技术是通往其他C语言艰深话题的门径。若不掌握，就算不上高级C程序员。当你自己也能写出这样的文档，并且有能力明智地评价它之后，才称得上C语言大师。

## 缘何写作本文 ##

2013年底，我大量应用了一项C语言优化技术，这项技术是我早在二十余年前就已掌握的，但彼时之后，鲜有使用。

我需要减少一个程序对内存空间的占用，它使用了上千（有时甚至几十万）C结构体实例。这个程序是cvs-fast-export，在将其应用于大规模软件仓库时，程序会出现内存耗尽错误。

通过精心调整结构成体员的顺序，可以在这种情况下大幅减少内存占用。其效果显著——在上述案例中，可以减少40%的内存空间。程序应用于更大的软件仓库，也不会因内存耗尽而崩溃。

但随着工作展开，我意识到这项技术在近些年几乎已被遗忘。Web搜索证实了我的想法，现今的C程序员们似乎已不再谈论这些话题，至少从搜索引擎中看不到。维基百科有些条目涉及这一主题，但未曾有人完整阐述。

事出有因。计算机科学课程（正确地）引导人们远离微观优化，转而寻求更理想的算法。计算成本一路走低，令压榨内存的必要性变得越来越低。旧日里，黑客们通过在陌生的硬件架构中跌跌撞撞学习——如今已不多见。

然而这项技术在关键时刻仍颇具价值，并且只要内存容量有限，价值就始终存在。本文意在节省C程序员重新发掘这项技术所需的时间，让他们有精力关注更重要任务。

## 对齐要求 ##

首先需要了解的是，对于现代处理器，C编译器在内存中放置基本C数据类型的方式受到约束，以令内存的访问速度更快。

在x86或ARM处理器中，基本C数据类型通常并不存储于内存中的随机字节地址。实际情况是，除char外，所有其他类型都有“对齐要求”：char可起始于任意字节地址，2字节的short必须从偶数字节地址开始，4字节的int或float必须从能被4整除的地址开始，8字节的long和double必须从能被8整除的地址开始。无论signed（有符号）还是unsigned（无符号）都不受影响。

用行话来说，x86和ARM上的基本C类型是“自对齐（self-aligned）”的。关于指针，无论32位（4字节）还是64位（8字节）也都是自对齐的。

自对齐可令访问速度更快，因为它有利于生成单指令（single-instruction）存取这些类型的数据。另一方面，如若没有对齐约束，可能最终不得不通过两个或更多指令访问跨越机器字边界的数据。字符数据是种特殊情况，因其始终处在单一机器字中，所以无论存取何处的字符数据，开销都是一致的。这也就是它不需要对齐的原因。

我提到“现代处理器”，是因为有些老平台强迫C程序违反对齐规则（例如，为int指针分配一个奇怪的地址并试图使用它），不仅令速度减慢，还会导致非法指令错误。例如Sun SPARC芯片就有这种问题。事实上，如果你下定决心，并恰当地在处理器中设置标志位（e18），在x86平台上，也能引发这种错误。

另外，自对齐并非唯一规则。纵观历史，有些处理器，由其是那些缺乏桶式移位器（Barrel shifter）的处理器限制更多。如果你从事嵌入式系统领域编程，有可能掉进这些潜伏于草丛之中的陷阱。小心这种可能。

你还可以通过pragma指令（通常为`#pragma pack`）强迫编译器不采用处理器惯用的对齐规则。但请别随意运用这种方式，因为它强制生成开销更大、速度更慢的代码。通常，采用我在下文介绍的方式，可以节省相同或相近的内存。

使用#pragma pack的唯一理由是——假如你需让C语言的数据分布，与某种位级别的硬件或协议完全匹配（例如内存映射硬件端口），而违反通用对齐规则又不可避免。如果你处于这种困境，且不了解我所讲述的内容，那你已深陷泥潭，祝君好运。

## 填充 ##

我们来看一个关于变量在内存中分布的简单案例。思考形式如下的一系列变量声明，它们处在一个C模块的顶层。

~~~C
char *p;
char c;
int x;
~~~

假如你对数据对齐一无所知，也许以为这3个变量将在内存中占据一段连续空间。也就是说，在32位系统上，一个4字节指针之后紧跟着1字节的char，其后又紧跟着4字节int。在64位系统中，唯一的区别在于指针将占用8字节。

然而实际情况（在x86、ARM或其他采用自对齐类型的平台上）如下。存储p需要自对齐的4或8字节空间，这取决于机器字的大小。这是指针对齐——极其严格。

c紧随其后，但接下来x的4字节对齐要求，将强制在分布中生成了一段空白，仿佛在这段代码中插入了第四个变量，如下所示。

~~~C
char *p;      /* 4 or 8 bytes */
char c;       /* 1 byte */
char pad[3];  /* 3 bytes */
int x;        /* 4 bytes */
~~~

字符数组`pad[3]`意味着在这个结构体中，有3个字节的空间被浪费掉了。老派术语将其称之为“废液（slop）”。

如果x为2字节short：

~~~C
char *p;
char c;
short x;
~~~

在这个例子中，实际分布将会是：

~~~C
char *p;      /* 4 or 8 bytes */
char c;       /* 1 byte */
char pad[1];  /* 1 byte */
short x;      /* 2 bytes */
~~~

另一方面，如果x为64位系统中的long：

~~~C
char *p;
char c;
long x;
~~~

我们将得到：

~~~C
char *p;     /* 8 bytes */
char c;      /* 1 byte */
char pad[7]; /* 7 bytes */
long x;      /* 8 bytes */
~~~

若你一路仔细读下来，现在可能会思索，何不首先声明较短的变量？

~~~C
char c;
char *p;
int x;
~~~

假如实际内存分布可以写成下面这样：

~~~C
char c;
char pad1[M];
char *p;
char pad2[N];
int x;
~~~

那`M`与`N`分别为几何？

首先，在此例中，`N`将为0，`x`的地址紧随`p`之后，能确保是与指针对齐的，因为指针的对齐要求总比int严格。

`M`的值就不易预测了。编译器若是恰好将`c`映射为机器字的最后一个字节，那么下一个字节（`p`的第一个字节）将恰好由此开始，并恰好与指针对齐。这种情况下，`M`将为0。

不过更有可能的情况是，`c`将被映射为机器字的首字节。于是乎`M`将会用于填充，以使`p`指针对齐——32位系统中为3字节，64位系统中为7字节。

中间情况也有可能发生。M的值有可能在0到7之间（32位系统为0到3），因为char可以从机器字的任何位置起始。

倘若你希望这些变量占用的空间更少，那么可以交换`x`与`c`的次序。

~~~C
char *p;     /* 8 bytes */
long x;      /* 8 bytes */
char c;      /* 1 byte */
~~~

通常，对于C代码中的少数标量变量（scalar variable），采用调换声明次序的方式能节省几个有限的字节，效果不算明显。而将这种技术应用于非标量变量（nonscalar variable）——尤其是结构体，则要有趣多了。

在讲述这部分内容前，我们先对标量数组做个说明。在具有自对齐类型的平台上，char、short、int、long和指针数组都没有内部填充，每个成员都与下一个成员自动对齐。

在下一节我们将会看到，这种情况对结构体数组并不适用。

## 结构体的对齐和填充 ##

通常情况下，结构体实例以其最宽的标量成员为基准进行对齐。编译器之所以如此，是因为此乃确保所有成员自对齐，实现快速访问最简便的方法。

此外，在C语言中，结构体的地址，与其第一个成员的地址一致——不存在头填充（leading padding）。小心：在C++中，与结构体相似的类，可能会打破这条规则！（是否真的如此，要看基类和虚拟成员函数是如何实现的，与不同的编译器也有关联。）

假如你对此有疑惑，ANSI C提供了一个`offsetof()`宏，可用于读取结构体成员位移。

考虑这个结构体：

~~~C
struct foo1 {
    char *p;
    char c;
    long x;
};
~~~

假定处在64位系统中，任何`struct fool`的实例都采用8字节对齐。不出所料，其内存分布将会像下面这样：

~~~C
struct foo1 {
    char *p;     /* 8 bytes */
    char c;      /* 1 byte */
    char pad[7]; /* 7 bytes */
    long x;      /* 8 bytes */
};
~~~

看起来仿佛与这些类型的变量单独声明别无二致。但假如我们将`c`放在首位，就会发现情况并非如此。

~~~C
struct foo2 {
    char c;      /* 1 byte */
    char pad[7]; /* 7 bytes */
    char *p;     /* 8 bytes */
    long x;      /* 8 bytes */
};
~~~

如果成员是互不关联的变量，`c`便可能从任意位置起始，`pad`的大小则不再固定。因为`struct foo2`的指针需要与其最宽的成员为基准对齐，这变得不再可能。现在`c`需要指针对齐，接下来填充的7个字节被锁定了。

现在，我们来谈谈结构体的尾填充（trailing padding）。为了解释它，需要引入一个基本概念，我将其称为结构体的“跨步地址（stride address）”。它是在结构体数据之后，与结构体对齐一致的首个地址。

结构体尾填充的通用法则是：编译器将会对结构体进行尾填充，直至它的跨步地址。这条法则决定了`sizeof()`的返回值。

考虑64位x86或ARM系统中的这个例子：

~~~C
struct foo3 {
    char *p;     /* 8 bytes */
    char c;      /* 1 byte */
};

struct foo3 singleton;
struct foo3 quad[4];
~~~

你以为`sizeof(struct foo3)`的值是9，但实际是16。它的跨步地址是`(&p)[2]`。于是，在`quad`数组中，每个成员都有7字节的尾填充，因为下个结构体的首个成员需要在８字节边界上对齐。内存分布就好像这个结构是这样声明的：

~~~C
struct foo3 {
    char *p;     /* 8 bytes */
    char c;      /* 1 byte */
    char pad[7];
};
~~~

作为对比，思考下面的例子：

~~~C
struct foo4 {
    short s;     /* 2 bytes */
    char c;      /* 1 byte */
};
~~~

因为`s`只需要2字节对齐，跨步地址仅在`c`的1字节之后，整个`struct foo4`也只需要1字节的尾填充。形式如下：

~~~C
struct foo4 {
    short s;     /* 2 bytes */
    char c;      /* 1 byte */
    char pad[1];
};
~~~

`sizeof(struct foo4)`的返回值将为4。

现在我们考虑位域（bitfields）。利用位域，你能声明比字符宽度更小的成员，低至１位，例如：

~~~C
struct foo5 {
    short s;
    char c;
    int flip:1;
    int nybble:4;
    int septet:7;
};
~~~

关于位域需要了解的是，它们是由字（或字节）层面的掩码和移位指令实现的。从编译器的角度来看，`struct foo5`中的位域就像２字节、16位的字符数组，只用到了其中12位。为了使结构体的长度是其最宽成员长度`sizeof(short)`的整数倍，接下来进行了填充。

~~~C
struct foo5 {
    short s;       /* 2 bytes */
    char c;        /* 1 byte */
    int flip:1;    /* total 1 bit */
    int nybble:4;  /* total 5 bits */
    int septet:7;  /* total 12 bits */
    int pad1:4;    /* total 16 bits = 2 bytes */
    char pad2;     /* 1 byte */
};
~~~

这是最后一个重要细节：如果你的结构体中含有结构体成员，内层结构体也要和最长的标量有相同的对齐。假如你写下了这段代码：

~~~C
struct foo6 {
    char c;
    struct foo5 {
        char *p;
        short x;
    } inner;
};
~~~

内层结构体成员`char *p`强迫外层结构体与内层结构体指针对齐一致。在64位系统中，实际的内存分布将类似这样：

~~~C
struct foo6 {
    char c;           /* 1 byte */
    char pad1[7];     /* 7 bytes */
    struct foo6_inner {
        char *p;      /* 8 bytes */
        short x;      /* 2 bytes */
        char pad2[6]; /* 6 bytes */
    } inner;
};
~~~

它启示我们，能通过重新打包节省空间。24个字节中，有13个为填充，浪费了超过50%的空间！

## 结构体成员重排 ##

理解了编译器在结构体中间和尾部插入填充的原因与方式后，我们来看看如何榨出这些废液。此即结构体打包的技艺。

首先注意，废液只存在于两处。其一是较大的数据类型（需要更严格的对齐）跟在较小的数据类型之后。其二是结构体自然结束的位置在跨步地址之前，这里需要填充，以使下个结构体能正确地对齐。

消除废液最简单的方式，是按对齐值递减重新对结构体成员排序。即让所有指针对齐成员排在最前面，因为在64位系统中它们占用8字节；然后是4字节的int；再然后是2字节的short，最后是字符。

因此，以简单的链表结构体为例：

~~~C
struct foo7 {
    char c;
    struct foo7 *p;
    short x;
};
~~~

将隐含的废液写明，形式如下：

~~~C
struct foo7 {
    char c;         /* 1 byte */
    char pad1[7];   /* 7 bytes */
    struct foo7 *p; /* 8 bytes */
    short x;        /* 2 bytes */
    char pad2[6];   /* 6 bytes */
};
~~~

总共是24字节。如果按长度重排，我们得到：

~~~C
struct foo8 {
    struct foo8 *p;
    short x;
    char c;
};
~~~

考虑到自对齐，我们看到所有数据域之间都不需填充。因为有较严对齐要求（更长）成员的跨步地址对不太严对齐要求的（更短）成员来说，总是合法的对齐地址。重打包过的结构体只需要尾填充：

~~~C
struct foo8 {
    struct foo8 *p; /* 8 bytes */
    short x;        /* 2 bytes */
    char c;         /* 1 byte */
    char pad[5];    /* 5 bytes */
};
~~~

重新打包将空间降为16字节。也许看起来不算很多，但假如这个链表的长度有20万呢？将会积少成多。

注意，重新打包不能确保在所有情况下都能节省空间。将这项技术应用于更靠前`struct foo6`的那个例子，我们得到：

~~~C
struct foo9 {
    struct foo9_inner {
        char *p;      /* 8 bytes */
        int x;        /* 4 bytes */
    } inner;
    char c;           /* 1 byte */
};
~~~

将填充写明：

~~~C
struct foo9 {
    struct foo9_inner {
        char *p;      /* 8 bytes */
        int x;        /* 4 bytes */
        char pad[4];  /* 4 bytes */
    } inner;
    char c;           /* 1 byte */
    char pad[7];      /* 7 bytes */
};
~~~

结果还是24字节，因为`c`无法作为内层结构体的尾填充。要想节省空间，你需要得新设计数据结构。

## 棘手的标量案例 ##

只有在符号调试器能显示枚举类型的名称而非原始整型数字时，使用枚举来代替`#define`才是个好办法。然而，尽管枚举必定与某种整型兼容，但Ｃ标准却没有指明究竟是何种底层整型。

请当心，重打包结构体时，枚举型变量通常是int，这与编译器相关；但也可能是short、long、甚至默认为char。编译器可能会有progma预处理指令或命令行选项指定枚举的尺寸。

`long double`是个类似的故障点。有些C平台以80位实现，有些是128位，还有些80位平台将其填充到96或128位。

以上两种情况，最好用`sizeof()`来检查存储尺寸。

最后，在x86 Linux系统中，double有时会破自对齐规则的例；在结构体内，8字节的double可能只要求4字节对齐，而在结构体外，独立的double变量又是8字节自对齐。这与编译器和选项有关。

## 可读性与缓存局部性 ##

尽管按尺寸重排是最简单的消除废液的方式，却不一定是正确的方式。还有两个问题需要考量：可读性与缓存局部性。

程序不仅与计算机交流，还与其他人交流。甚至（尤其是！）交流的对象只有将来你自己时，代码可读性依然重要。

笨拙地、机械地重排结构体可能有损可读性。倘若有可能，最好这样重排成员：将语义相关的数据放在一起，形成连贯的组。最理想的情况是，结构体的设计应与程序的设计相通。

当程序频繁访问某一结构体或其一部分时，若能将其放入一个缓存段，对提高性能颇有帮助。缓存段是这样的内存块——当处理器获取内存中的任何单个地址时，会把整块数据都取出来。　在64位x86上，一个缓存段为64字节，它开始于自对齐的地址。其他平台通常为32字节。

为保持可读性所做的工作（将相关和同时访问的数据放在临近位置）也会提高缓存段的局部性。这些都是需要明智地重排，并对数据的存取模式了然于心的原因。

如果代码从多个线程并发访问同一结构体，还存在第三个问题：缓存段弹跳（cache line bouncing）。为了尽量减少昂贵的总线通信，应当这样安排数据——在一个更紧凑的循环里，从一个缓存段中读数据，而向另一个写入数据。

是的，某些时候，这种做法与前文将相关数据放入与缓存段长度相同块的做法矛盾。多线程的确是个难题。缓存段弹跳和其他多线程优化问题是很高级的话题，值得单独为它们写份指导。这里我所能做的，只是让你了解有这些问题存在。

## 其他打包技术 ##

在为结构体瘦身时，重排序与其他技术结合在一起效果最好。例如结构体中有几个布尔标志，可以考虑将其压缩成1位的位域，然后把它们打包放在原本可能成为废液的地方。

你可能会有一点儿存取时间的损失，但只要将工作集合压缩得足够小，那点损失可以靠避免缓存未命中补偿。

更通用的原则是，选择能把数据类型缩短的方法。以cvs-fast-export为例，我使用的一个压缩方法是：利用RCS和CVS在1982年前还不存在这个事实，我弃用了64位的Unix`time_t`（在1970年开始为零），转而用了一个32位的、从1982-01-01T00:00:00开始的偏移量；这样日期会覆盖到2118年。（注意：若使用这类技巧，要用边界条件检查以防讨厌的Bug！）

这不仅减小了结构体的可见尺寸，还可以消除废液和/或创造额外的机会来进行重新排序。这种良性串连的效果不难被触发。

最冒险的打包方法是使用union。假如你知道结构体中的某些域永远不会跟另一些域共同使用，可以考虑用union共享它们存储空间。不过请特别小心并用回归测试验证。因为如果分析出现一丁点儿错误，就会引发从程序崩溃到微妙数据损坏（这种情况糟得多）间的各种错误。

## 工具 ##

clang编译器有个Wpadded选项，可以生成有关对齐和填充的信息。

还有个叫pahole的工具，我自己没用过，但据说口碑很好。该工具与编译器协同工作，生成关于结构体填充、对齐和缓存段边界报告。

## 证明和例外 ##

读者可以下载一段程序源代码[packtest.c](http://www.catb.org/esr/structure-packing/packtest.c)，验证上文有关标量和结构体尺寸的结论。

如果你仔细检查各种编译器、选项和罕见硬件的稀奇组合，会发现我前面提到的部分规则存在例外。越早期的处理器设计例外越常见。

理解这些规则的第二个层次是，知其何时及如何会被打破。在我学习它们的日子里（1980年代早期），我们把不理解这些规则的人称为“所有机器都是VAX综合症”的牺牲品。记住，世上所有电脑并非都是PC。
