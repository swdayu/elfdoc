x64 补充规范

* `底层机器接口`_

  * `处理器架构`_
  * `数据表示`_
  * `特殊类型`_
  * `复合类型`_

AMD64 架构是对 x86 架构的扩展，任何实现 AMD64 架构规范的处理器也提供对 Intel 8086 架
构的兼容，包括32位处理器，如 Intel 386、Intel Pentium、AMD K6-2。遵循 AMD64 ABI 的操
作系统可能提供在这些兼容模式下运行的支持。但该文档介绍的 AMD64 ABI 不适用于这种模式，仅
适用于在 AMD64 架构提供的长模式下运行的程序。

使用 AMD64 指令集的二进制文件可以针对32位模型进行编程，在该模型中 C 数据类型 int、long、
和所有的指针都是 32 位对象（ILP32），或针对64位模型编程，在该模型中 C 数据类型 int 是
32 位，但 long 和所有指针类型都是 64 位对象（LP64）。本规范涵盖了 LP64 和 ILP32 编程
模式。

除非另有说明，AMD64 架构 ABI 遵循 Intel386 ABI 中描述的约定。AMD64 ABI 并没有复制整个
Intel386 ABI 的内容，而是仅指出对 Intel386 ABI 进行了哪些修改。本文档并未尝试为除 C 语
言之外的其他编程语言指定 ABI。然而，可以假设有很多编程语言都会与用 C 语言编写的代码链接，
因此这里记录的 ABI 规范也适用。AMD64 由于历史原因原来也被称为 x86-64。

底层机器接口
=============

处理器架构
----------

任何程序都可以预期 AMD64 处理器实现了下表中列出的基准功能。大多数特性的名称都与 CPUID
的比特位对应，可参考处理器手册。除了 OSFXSR 和 SCE 例外，它们由 %cr4 以及 IA32_EFER
MSR 寄存器中的位控制。

除了 AMD64 基准架构外，还定义了由后续 CPU 模块实现的微架构级别，这些级别从 x86-64-v2
开始。这些级别在支持它们的系统上可以利用其中的特性来优化程序性能，而且级别是累加的，即前
一级别的特性在后续级别中是隐含的。

x86-64-v3 和 x86-64-v4 级别只有在相应特性已经完全启用时才可用，这意味着系统必须通过处
理器手册中对这些特性的完整检查，包括使用 xgetbv 获得的 XCR0 特性标志的验证。 ::

    级别名称        CPU特性             示例指令
    基准            CMOV                cmov
                    CX8                 cmpxchg8b
                    FPU                 fld
                    FXSR                fxsave
                    MMX                 emms
                    OSFXSR              fxsave
                    SCE                 syscall
                    SSE                 cvtss2si
                    SSE2                cvtpi2pd
    x86-64-v2       CMPXCHG16B          cmpxchg16b
                    LAHF-SAHF           lahf
                    POPCNT              popcnt
                    SSE3                addsubpd
                    SSE4_1              blendpd
                    SSE4_2              pcmpestri
                    SSSE3               phaddd
    x86-64-v3       AVX                 vzeroall
                    AVX2                vpermd
                    BMI1                andn
                    BMI2                bzhi
                    F16C                vcvtph2ps
                    FMA                 vfmaddl32pd
                    LZCNT               lzcnt
                    MOVBE               movbe
                    OSXSAVE             xgetbv
    x86-64-v4       AVX512F             kmovw
                    AVX512BW            vdbpsadbw
                    AVX512CD            vplzcntd
                    AVX512DQ            vpmullq
                    AVX512VL            n/a

上表中的微架构级别名称被用作目录名称，动态链接器会根据当前 CPU 支持的级别来搜索这些目录，
编译器也会根据这些名称来选择 CPU 的一组特性。发行版本也可以指定需要执行的处理器级别。

例如，要选择级别 x86-64-v3，程序员会使用 -march=x86-64-v3 GCC 标志构建一个共享对象。
生成的共享对象需要安装在目录 /usr/lib64/glibc-hwcaps/x86-64-v3 或
/usr/lib/x86_64-linux-gnu/glibc-hwcaps/x86-64-v3（对于多架构文件系统布局的发行版）。
为了支持仅实现 K8 的基准功能，必须将后备版本安装到默认位置 /usr/lib64 或
/usr/lib/x86_64-linux-gnu。它必须使用 -march=x86-64（GCC 默认值）构建。如果不遵循这
一指导原则，在不支持这一级别的系统上加载库会失败。

安装在匹配的 glibc-hwcaps 子目录下的共享对象可以使用此级别及以前级别的 CPU 特性，而无
需进一步的检测逻辑。对于本节未列出的其他 CPU 特性，或者仅在更高级别下列出的 CPU 特性，
仍然需要运行时检测（即使所有当前 CPU 都实现了某些 CPU 特性）。

如果发行版需要支持某个特定级别，他们会使用适当的 -march= 选项构建所有内容，并将构建的二
进制文件安装在默认的文件系统位置。针对此类发行版时，程序员可以使用相同的 -march= 选项构
建他们的二进制文件，并将它们安装在默认位置。针对更高级别的优化共享对象仍然可以安装到具有
适当名称的子目录中。

数据表示
---------

基本类型： ::

    C 语言类型                          大小和对齐  AMD64架构
    char/signed/unsigned                1   1       byte
    short/signed/unsigned               2   2       twobyte
    int/signed/enum†††/unsigned         4   4       fourbyte
    long/signed/unsigned (ILP32)        4   4       fourbyte
    long/signed/unsigned (LP64)         8   8       eightbyte
    long long/signed/unsigned           8   8††††   eightbyte
    __int128/signed/unsigned††          16  16      sixteenbyte
    any-type*/any-type(*)() (ILP32)     4   4       fourbyte
    any-type*/any-type(*)() (LP64)      8   8       eightbyte
    _Float16††                          2   2       16-bit (IEEE-754)
    float                               4   4       single (IEEE-754)
    double                              8   8††††   double (IEEE-754)
    __float80††/long double†††††        16  16      80-bit extended (IEEE-754)
    __float128††/long double†††††       16  16      128-bit extended (IEEE-754)
    _Decimal32                          4   4       32-bit BID (IEEE-754R)
    _Decimal64                          8   8       64-bit BID (IEEE-754R)
    _Decimal128                         16  16      128-bit BID (IEEE-754R)
    __m64††                             8   8       MMX and 3DNow!
    __m128††                            16  16      SSE and SSE-2
    __m256††                            32  32      AVX
    __m512††                            64  64      AVX-512

    †† 对这些类型的支持是可选的，_Float16 来自于 ISO/IEC TS 18661-3:2015 标准
    ††† C++ 和一些 C 实现允许比 int 更长的类型作为枚举类型，在这种情况下，底层类型会依
        次提升为 int、unsigned int、long int 或 unsigned long int
    †††† Intel386 上 long long 和 double 是 4 字节对齐，AMD64 上是 8 字节对齐
    ††††† 类型 long double 在 Android 平台上与 __float128 相同，都是 16 字节

80位浮点使用15位指数（exponent），64位尾数（mantissa），最高有效位是隐含的，指数的偏移
量为 16383。128位浮点使用15位指数，113位尾数，最高有效位是隐含的，指数的偏移量为 16383。
AMD64 架构的最初实现仅通过软件仿真来支持128位浮点类型的操作。

浮点类型 __float80 和 long double 占有 16 字节，其中只有 10 字节是有效的，剩余 6 字节
是尾部填充，这些字节的内容未定义。__int128 在内存中适用小端字节序，即低64位在低地址，高
64位在高地址。size_t 在 ILP32 模式定义为 unsigned int，在 LP64 模式定义为 unsigned
long。布尔类型如果保存在内存是一个字节0表示假1表示真，如果在整型寄存器中（除了作为参数传
递）整个8字节都是有效的任何非零值都表示真。

与 Intel386 一样，AMD64 不一定需要类型是严格对齐的，不对齐的基本类型仅仅是访问速度慢一
点，其他行为是相同的。但 __m128、__m256、__m512 这些类型例外，这些类型必须严格对齐。

特殊类型
---------

_BitInt(N) 是 ISO/IEC WG14 N2763 标准中定义的一组整数类型，其中 N 是表示类型位数的整
型常量表达式。_BitInt(N) 类型默认为有符号，unsigned _BitInt(N) 为无符号。_BitInt(N)
类型在内存中以小端序存储。每个字节中的位从右到左分配。对于 N <= 64 的情况，它们的大小和
对齐方式与能够容纳它们的最小（有符号和无符号）char、short、int、long 和 long long 类型
相同。对于 N > 64 的情况，它们被视为由 64 位整数块组成的结构体，块的数量是能够容纳该类
型的最小数量，_BitInt(N) 类型对齐到 64 位，这些类型的大小是大于或等于 N 的 64 位块的最
小整数倍数。当 _BitInt(N) 值存储在内存或寄存器中时，超出 _BitInt(N) 宽度但在 _BitInt(N)
大小范围内的未使用位的值是未指定的。该类型的数组也可以使用常见的 sizeof(Array)/sizeof(ElementType)
表达式。

__bf16 类型（该类型用于 BF16 指令）是 16 位浮点数的另一种编码格式，具有 8 位指数和 7
位尾数。__bf16 代表 Brain Floating Point Format（该类型用于加速机器学习特别是深度学习
训练算法），这是 32 位 IEEE 754 单精度浮点格式的截断（16 位）版本。它与 _Float16 具有
相同的大小、对齐方式、参数传递和返回规则。

复合类型
---------

结构体和联合体的对齐字节是其最严格对齐成员的对齐字节数。每个成员都被分配到具有适当对齐的
最低可用偏移位置。任何对象的大小始终是其对齐字节的倍数。结构体和联合体对象可能需要填充以
满足大小和对齐约束。任何填充的内容是未定义的。

数组使用与其元素相同的对齐方式，除非局部或全局数组变量的长度至少为 16 字节，或者 C99 变
长数组变量始终至少有 16 字节的对齐。数组的这种对齐要求允许在操作数组时使用 SSE 指令。编
译器通常无法计算变长数组（VLA）的大小，但预计大多数 VLA 将需要至少 16 字节，因此规定
VLA 至少有 16 字节的对齐是合理的。

C 语言中的结构体和联合体定义可以包含位域，这些位域定义了指定比特位宽的整数值。应用程序二
进制接口（ABI）不允许位域具有类型 __m64、__m128、__m256 或 __m512。使用这些类型的位域
的程序不具备可移植性。

既没指定 signed 也没指定 unsigned 的位域始终具有非负值，尽管这些类型 char、short、int
或 long 可能有负值，但这些位域的范围与相应无符号类型的位域相同。位域遵循与其他结构体和联
合体成员相同的大小和对齐规则。此外，位域是从右到左分配，位域必须包含在适合其声明类型的存
储单元中，位域可以与其他结构体/联合体成员共享存储单元。未命名位域的类型不影响结构体或联
合体的对齐。

函数调用约定
=============



操作系统接口
============



代码示例
=========



目标文件
=========




动态链接
=========



