https://www.sco.com/developers/gabi/latest/contents.html

通用格式规范（2013年6月10日）

* `目标文件`_

  * `文件格式`_
  * `数据表示`_
  * `文件头部`_
  * `文件标识信息`_
  * `程序分区`_
  * `分区链接规则`_
  * `分区分组`_
  * `特殊分区`_
  * `字符串表分区`_
  * `符号表分区`_
  * `重定位分区`_
  * `重定位类型`_

* `程序加载`_

  * `程序头部`_
  * `基地址`_
  * `分段权限`_
  * `分段内容`_
  * `说明分段`_
  * `TLS 分段`_
  * `加载程序`_

* `动态链接`_

  * `程序解释器`_
  * `动态链接器`_
  * `动态链接段`_
  * `共享目标依赖`_
  * `替换序列`_
  * `全局偏移表`_
  * `过程链接表`_
  * `哈希表`_
  * `初始和终止函数`_

编译的应用程序二进制接口由两个基本部分组成，其中通用规范（gABI, generic ABI）是独立于
不同架构的不变部分，另一部分是特定处理器补充规范（psABI, processor supplement ABI）。
这里定义的是 ELF 文件格式的通用规范部分。

目标文件
=========

本章描述 ELF 目标文件格式，有三种主要的目标文件类型：

1. 可重定位文件（relocatable file）包含代码和数据，可以与其他目标文件一起链接成可执行
   文件或共享目标文件；

2. 可执行文件（executable file）包含一个可执行程序，文件指定了过程 exec(BA_OS) 怎样创
   建程序的进程映像；

3. 共享目标文件（shared object file）包含代码和数据，可用于两种情况下的链接，第一种情
   况是可以与其他可重定位文件或共享目标文件一起链接成新的目标文件，第二种情况是动态链接
   器将该文件绑定到一个可执行文件或其他共享目标来创建一个新的进程映像；

目标文件是由汇编器或者链接器创建的，能直接在处理器上执行的程序二进制表示。

文件格式
---------

目标文件不仅涉及到程序链接（构建程序），还涉及到程序执行（运行程序）。因此为了方便和高
效，目标文件格式提供了文件内容的并行视图，反映这两种行为的不同。下图是一个目标文件的组织
方式： ::

    链接视图                       执行视图
    ------------------------      -------------------------
    ELF Header                    ELF Header
    Program Header Table (O)      Program Header Table (R)
    Section 1                     Segment 1
    Section 2                     Segment 2
    ...                           ...
    Section n                     Segment n
    Section Header Table (R)      Section Header Table (O)

    (O) 可选（optional）
    (R) 必须存在（required）

ELF 头部位于文件开始处，包含了一个路径地图来描述文件组织方式。分区包含了用于连接视图的
一块目标文件信息，包括：指令、数据、符号表、可重定位信息等等，还存在一些特殊的分区。另
外，分段和文件的执行视图会在后面的章节介绍。

一个程序头部表告诉系统怎样创建进程映像，用于执行的目标文件必须包含一个程序头部表，而可重
定位文件不需要。分区头部表包含了文件的分区信息，每个分区在这个表中都有一个条目，每一个条
目指定了分区的一些信息，比如分区名称、分区大小等等。用于链接的必须要有一个分区头部表，其
他目标文件可以有也可以没有分区表。

尽管图中显示程序头部表紧跟在 ELF 头部后面，分区头部表紧跟在分区后面，但是实际的文件可能
有所不同。而且，分区和分段没有特别的顺序。只有 ELF 头部的在文件种的位置是固定的。

数据表示
---------

ELF 支持一系列 32-bit 或 64-bit 平台的数据类型，这些类型分别对齐到自己大小的地址边界，
例如 8 字节大小的类型对齐到 8 字节地址边界，4 字节大小的类型对齐到 4 字节地址边界。为了
移植方便，ELF 不使用位域。 ::

    32位机器数据类型    类型长度    对齐要求
    Elf32_Addr          4           4           无符号程序地址
    Elf32_Off           4           4           无符号文件偏移
    Elf32_Half          2           2           无符号半整型
    Elf32_Word          4           4           无符号整型
    Elf32_Sword         4           4           有符号整型
    Elf32_Byte          1           1           无符号字节类型

    64位机器数据类型    类型长度    对齐要求
    Elf64_Addr          8           8           无符号程序地址
    Elf64_Off           8           8           无符号文件偏移
    Elf64_Half          2           2           无符号半整型
    Elf64_Word          4           4           无符号整型
    Elf64_Sword         4           4           有符号整型
    Elf64_Xword         8           8           无符号长整型
    Elf64_Sxword        8           8           有符号长整型
    Elf64_Byte          1           1           无符号字节类型

文件头部
---------

目标文件的一些控制结构的大小可能会发生变化，因为头部信息包含了这些结构的真实大小；如果程
序遇到一个控制结构的大小比预期的更大或更小，程序可能忽略额外的部分，而对于丢失的部分的处
理，依赖于当时的上下文以及扩展的设定（如果使用了扩展）。

ELF 文件头部结构体： ::

    #define EI_NIDENT 16

    typedef struct {
        Elf32_Byte  e_ident[EI_NIDENT];
        Elf32_Half  e_type;
        Elf32_Half  e_machine;
        Elf32_Word  e_version;
        Elf32_Addr  e_entry;
        Elf32_Off   e_phoff;
        Elf32_Off   e_shoff;
        Elf32_Word  e_flags;
        Elf32_Half  e_ehsize;
        Elf32_Half  e_phentsize;
        Elf32_Half  e_phnum;
        Elf32_Half  e_shentsize;
        Elf32_Half  e_shnum;
        Elf32_Half  e_shstrndx;
    } Elf32_Ehdr;

    typedef struct {
        Elf32_Byte  e_ident[EI_NIDENT];
        Elf64_Half  e_type;
        Elf64_Half  e_machine;
        Elf64_Word  e_version;
        Elf64_Addr  e_entry;
        Elf64_Off   e_phoff;
        Elf64_Off   e_shoff;
        Elf64_Word  e_flags;
        Elf64_Half  e_ehsize;
        Elf64_Half  e_phentsize;
        Elf64_Half  e_phnum;
        Elf64_Half  e_shentsize;
        Elf64_Half  e_shnum;
        Elf64_Half  e_shstrndx;
    } Elf64_Ehdr;

e_ident
    ELF `文件标识信息`_ （Elf Identification）
e_type
    目标文件类型
e_machine
    处理器信息
e_version
    ELF 文件格式版本，当前只有一个版本 EV_CURRENT
e_entry
    程序入口的虚拟地址，如果没有入口，该成员为 0
e_phoff
    程序头部表的文件偏移，如果没有程序头部表，该成员为 0
e_shoff
    分区头部表的文件偏移，如果没有分区头部表，该成员为 0
e_flags
    处理器标记
e_ehsize
    ELF 头部大小
e_phentsize
    程序头部的大小，程序头部表由程序头部组成，每个条目的大小相同
e_phnum
    程序头部表条目的个数
e_shentsize
    分区头部的大小，分区头部表由分区头部组成，每个条目的大小相同
e_shnum
    分区头部表中分区头部的个数，如果个数大于等于 SHN_LORESERVE（0xff00），这个成员的
    值为 0，具体的分区头部的个数位于第一个分区头部的 sh_size 变量中，否则第一个分区头
    部的 sh_size 值为 0
e_shstrndx
    分区名称字符串表位于哪个分区头部，如果没有分区名称字符串表，该成员的值为 SHN_UNDEF；
    如果该值大于等于 SHN_LORESERVE（0xff00），该成员的值应该设为 SHN_XINDEX（0xffff），
    具体的分区索引值位于第一个分区头部的 sh_link 变量中，否则第一个分区头部的 link 值
    为 0

目标文件类型： ::

    ET_NONE     0       No file type
    ET_REL      1       Relocatable file
    ET_EXEC     2       Executable file
    ET_DYN      3       Shared object file
    ET_CORE     4       Core file
    ET_LOOS     0xfe00  Operating system-specific
    ET_HIOS     0xfeff  Operating system-specific
    ET_LOPROC   0xff00  Processor-specific
    ET_HIPROC   0xffff  Processor-specific

ET_NONE
    未知文件类型
ET_REL
    可重定位文件
ET_EXEC
    可执行文件
ET_DYN
    共享目标文件
ET_CORE
    核心文件类型，当前没有定义该类型的文件内容
ET_LOOS ET_HIOS
    操作系统预留
ET_LOPROC ET_HIPROC
    处理器预留

处理器信息指定当前文件包含的内容相对于哪个处理器架构，处理器补充规范定义的 ELF 名字使用
处理器名称来区分，例如处理器标记 ``EF_``，名为 ``WIDGET`` 的标记相对于处理器 ``EM_XYZ``
的名称为 ``EF_XYZ_WIDGET``。 ::

    EM_NONE             0 No machine
    EM_M32              1 AT&T WE 32100
    EM_SPARC            2 SPARC
    EM_386              3 Intel 80386
    EM_68K              4 Motorola 68000
    EM_88K              5 Motorola 88000
    EM_IAMCU            6 Intel MCU
    EM_860              7 Intel 80860
    EM_MIPS             8 MIPS I Architecture
    EM_S370             9 IBM System/370 Processor
    EM_MIPS_RS3_LE      10 MIPS RS3000 Little-endian
    reserved            11-14 Reserved for future use
    EM_PARISC           15 Hewlett-Packard PA-RISC
    reserved            16 Reserved for future use
    EM_VPP500           17 Fujitsu VPP500
    EM_SPARC32PLUS      18 Enhanced instruction set SPARC
    EM_960              19 Intel 80960
    EM_PPC              20 PowerPC
    EM_PPC64            21 64-bit PowerPC
    EM_S390             22 IBM System/390 Processor
    EM_SPU              23 IBM SPU/SPC
    reserved            24-35 Reserved for future use
    EM_V800             36 NEC V800
    EM_FR20             37 Fujitsu FR20
    EM_RH32             38 TRW RH-32
    EM_RCE              39 Motorola RCE
    EM_ARM              40 ARM 32-bit architecture (AARCH32)
    EM_ALPHA            41 Digital Alpha
    EM_SH               42 Hitachi SH
    EM_SPARCV9          43 SPARC Version 9
    EM_TRICORE          44 Siemens TriCore embedded processor
    EM_ARC              45 Argonaut RISC Core, Argonaut Technologies Inc.
    EM_H8_300           46 Hitachi H8/300
    EM_H8_300H          47 Hitachi H8/300H
    EM_H8S              48 Hitachi H8S
    EM_H8_500           49 Hitachi H8/500
    EM_IA_64            50 Intel IA-64 processor architecture
    EM_MIPS_X           51 Stanford MIPS-X
    EM_COLDFIRE         52 Motorola ColdFire
    EM_68HC12           53 Motorola M68HC12
    EM_MMA              54 Fujitsu MMA Multimedia Accelerator
    EM_PCP              55 Siemens PCP
    EM_NCPU             56 Sony nCPU embedded RISC processor
    EM_NDR1             57 Denso NDR1 microprocessor
    EM_STARCORE         58 Motorola Star*Core processor
    EM_ME16             59 Toyota ME16 processor
    EM_ST100            60 STMicroelectronics ST100 processor
    EM_TINYJ            61 Advanced Logic Corp. TinyJ embedded processor family
    EM_X86_64           62 AMD x86-64 architecture
    EM_PDSP             63 Sony DSP Processor
    EM_PDP10            64 Digital Equipment Corp. PDP-10
    EM_PDP11            65 Digital Equipment Corp. PDP-11
    EM_FX66             66 Siemens FX66 microcontroller
    EM_ST9PLUS          67 STMicroelectronics ST9+ 8/16 bit microcontroller
    EM_ST7              68 STMicroelectronics ST7 8-bit microcontroller
    EM_68HC16           69 Motorola MC68HC16 Microcontroller
    EM_68HC11           70 Motorola MC68HC11 Microcontroller
    EM_68HC08           71 Motorola MC68HC08 Microcontroller
    EM_68HC05           72 Motorola MC68HC05 Microcontroller
    EM_SVX              73 Silicon Graphics SVx
    EM_ST19             74 STMicroelectronics ST19 8-bit microcontroller
    EM_VAX              75 Digital VAX
    EM_CRIS             76 Axis Communications 32-bit embedded processor
    EM_JAVELIN          77 Infineon Technologies 32-bit embedded processor
    EM_FIREPATH         78 Element 14 64-bit DSP Processor
    EM_ZSP              79 LSI Logic 16-bit DSP Processor
    EM_MMIX             80 Donald Knuth's educational 64-bit processor
    EM_HUANY            81 Harvard University machine-independent object files
    EM_PRISM            82 SiTera Prism
    EM_AVR              83 Atmel AVR 8-bit microcontroller
    EM_FR30             84 Fujitsu FR30
    EM_D10V             85 Mitsubishi D10V
    EM_D30V             86 Mitsubishi D30V
    EM_V850             87 NEC v850
    EM_M32R             88 Mitsubishi M32R
    EM_MN10300          89 Matsushita MN10300
    EM_MN10200          90 Matsushita MN10200
    EM_PJ               91 picoJava
    EM_OPENRISC         92 OpenRISC 32-bit embedded processor
    EM_ARC_COMPACT      93 ARC International ARCompact processor (old spelling/synonym: EM_ARC_A5)
    EM_XTENSA           94 Tensilica Xtensa Architecture
    EM_VIDEOCORE        95 Alphamosaic VideoCore processor
    EM_TMM_GPP          96 Thompson Multimedia General Purpose Processor
    EM_NS32K            97 National Semiconductor 32000 series
    EM_TPC              98 Tenor Network TPC processor
    EM_SNP1K            99 Trebia SNP 1000 processor
    EM_ST200            100 STMicroelectronics (www.st.com) ST200 microcontroller
    EM_IP2K             101 Ubicom IP2xxx microcontroller family
    EM_MAX              102 MAX Processor
    EM_CR               103 National Semiconductor CompactRISC microprocessor
    EM_F2MC16           104 Fujitsu F2MC16
    EM_MSP430           105 Texas Instruments embedded microcontroller msp430
    EM_BLACKFIN         106 Analog Devices Blackfin (DSP) processor
    EM_SE_C33           107 S1C33 Family of Seiko Epson processors
    EM_SEP              108 Sharp embedded microprocessor
    EM_ARCA             109 Arca RISC Microprocessor
    EM_UNICORE          110 Microprocessor series from PKU-Unity Ltd. and MPRC of Peking University
    EM_EXCESS           111 eXcess: 16/32/64-bit configurable embedded CPU
    EM_DXP              112 Icera Semiconductor Inc. Deep Execution Processor
    EM_ALTERA_NIOS2     113 Altera Nios II soft-core processor
    EM_CRX              114 National Semiconductor CompactRISC CRX microprocessor
    EM_XGATE            115 Motorola XGATE embedded processor
    EM_C166             116 Infineon C16x/XC16x processor
    EM_M16C             117 Renesas M16C series microprocessors
    EM_DSPIC30F         118 Microchip Technology dsPIC30F Digital Signal Controller
    EM_CE               119 Freescale Communication Engine RISC core
    EM_M32C             120 Renesas M32C series microprocessors
    reserved            121-130 Reserved for future use
    EM_TSK3000          131 Altium TSK3000 core
    EM_RS08             132 Freescale RS08 embedded processor
    EM_SHARC            133 Analog Devices SHARC family of 32-bit DSP processors
    EM_ECOG2            134 Cyan Technology eCOG2 microprocessor
    EM_SCORE7           135 Sunplus S+core7 RISC processor
    EM_DSP24            136 New Japan Radio (NJR) 24-bit DSP Processor
    EM_VIDEOCORE3       137 Broadcom VideoCore III processor
    EM_LATTICEMICO32    138 RISC processor for Lattice FPGA architecture
    EM_SE_C17           139 Seiko Epson C17 family
    EM_TI_C6000         140 The Texas Instruments TMS320C6000 DSP family
    EM_TI_C2000         141 The Texas Instruments TMS320C2000 DSP family
    EM_TI_C5500         142 The Texas Instruments TMS320C55x DSP family
    EM_TI_ARP32         143 Texas Instruments Application Specific RISC Processor, 32bit fetch
    EM_TI_PRU           144 Texas Instruments Programmable Realtime Unit
    reserved            145-159 Reserved for future use
    EM_MMDSP_PLUS       160 STMicroelectronics 64bit VLIW Data Signal Processor
    EM_CYPRESS_M8C      161 Cypress M8C microprocessor
    EM_R32C             162 Renesas R32C series microprocessors
    EM_TRIMEDIA         163 NXP Semiconductors TriMedia architecture family
    EM_QDSP6            164 QUALCOMM DSP6 Processor
    EM_8051             165 Intel 8051 and variants
    EM_STXP7X           166 STMicroelectronics STxP7x family of configurable and extensible RISC processors
    EM_NDS32            167 Andes Technology compact code size embedded RISC processor family
    EM_ECOG1            168 Cyan Technology eCOG1X family
    EM_ECOG1X           168 Cyan Technology eCOG1X family
    EM_MAXQ30           169 Dallas Semiconductor MAXQ30 Core Micro-controllers
    EM_XIMO16           170 New Japan Radio (NJR) 16-bit DSP Processor
    EM_MANIK            171 M2000 Reconfigurable RISC Microprocessor
    EM_CRAYNV2          172 Cray Inc. NV2 vector architecture
    EM_RX               173 Renesas RX family
    EM_METAG            174 Imagination Technologies META processor architecture
    EM_MCST_ELBRUS      175 MCST Elbrus general purpose hardware architecture
    EM_ECOG16           176 Cyan Technology eCOG16 family
    EM_CR16             177 National Semiconductor CompactRISC CR16 16-bit microprocessor
    EM_ETPU             178 Freescale Extended Time Processing Unit
    EM_SLE9X            179 Infineon Technologies SLE9X core
    EM_L10M             180 Intel L10M
    EM_K10M             181 Intel K10M
    reserved            182 Reserved for future Intel use
    EM_AARCH64          183 ARM 64-bit architecture (AARCH64)
    reserved            184 Reserved for future ARM use
    EM_AVR32            185 Atmel Corporation 32-bit microprocessor family
    EM_STM8             186 STMicroeletronics STM8 8-bit microcontroller
    EM_TILE64           187 Tilera TILE64 multicore architecture family
    EM_TILEPRO          188 Tilera TILEPro multicore architecture family
    EM_MICROBLAZE       189 Xilinx MicroBlaze 32-bit RISC soft processor core
    EM_CUDA             190 NVIDIA CUDA architecture
    EM_TILEGX           191 Tilera TILE-Gx multicore architecture family
    EM_CLOUDSHIELD      192 CloudShield architecture family
    EM_COREA_1ST        193 KIPO-KAIST Core-A 1st generation processor family
    EM_COREA_2ND        194 KIPO-KAIST Core-A 2nd generation processor family
    EM_ARC_COMPACT2     195 Synopsys ARCompact V2
    EM_OPEN8            196 Open8 8-bit RISC soft processor core
    EM_RL78             197 Renesas RL78 family
    EM_VIDEOCORE5       198 Broadcom VideoCore V processor
    EM_78KOR            199 Renesas 78KOR family
    EM_56800EX          200 Freescale 56800EX Digital Signal Controller (DSC)
    EM_BA1              201 Beyond BA1 CPU architecture
    EM_BA2              202 Beyond BA2 CPU architecture
    EM_XCORE            203 XMOS xCORE processor family
    EM_MCHP_PIC         204 Microchip 8-bit PIC(r) family
    EM_INTEL205         205 Reserved by Intel
    EM_INTEL206         206 Reserved by Intel
    EM_INTEL207         207 Reserved by Intel
    EM_INTEL208         208 Reserved by Intel
    EM_INTEL209         209 Reserved by Intel
    EM_KM32             210 KM211 KM32 32-bit processor
    EM_KMX32            211 KM211 KMX32 32-bit processor
    EM_KMX16            212 KM211 KMX16 16-bit processor
    EM_KMX8             213 KM211 KMX8 8-bit processor
    EM_KVARC            214 KM211 KVARC processor
    EM_CDP              215 Paneve CDP architecture family
    EM_COGE             216 Cognitive Smart Memory Processor
    EM_COOL             217 Bluechip Systems CoolEngine
    EM_NORC             218 Nanoradio Optimized RISC
    EM_CSR_KALIMBA      219 CSR Kalimba architecture family
    EM_Z80              220 Zilog Z80
    EM_VISIUM           221 Controls and Data Services VISIUMcore processor
    EM_FT32             222 FTDI Chip FT32 high performance 32-bit RISC architecture
    EM_MOXIE            223 Moxie processor family
    EM_AMDGPU           224 AMD GPU architecture
    reserved            225 - 242 
    EM_RISCV            243 RISC-V

目标文件版本： ::

    EV_NONE     0 未知版本
    EV_CURRENT  1 当前版本

文件标识信息
------------

文件头部的标识信息，用于指定文件怎样解析，对应信息的索引位置为： ::

    EI_MAG0~EI_MAG3 0~3 文件起始信息
    EI_CLASS        4   文件使用的数据类型
    EI_DATA         5   数据字节序
    EI_VERSION      6   文件版本
    EI_OSABI        7   使用的补充规范（Operating system/ABI identification）
    EI_ABIVERSION   8   补充规范的版本（ABI version），依赖于补充规范的定义，如果没有版本信息应该设置为 0
    EI_PAD          9   未使用字节的起始位置，这些是预留字节要设为全零
    EI_NIDENT       16  文件标识信息总大小

文件起始信息： ::

                值      位置
    ELFMAG0     0x7f    e_ident[EI_MAG0]
    ELFMAG1     'E'     e_ident[EI_MAG1]
    ELFMAG2     'L'     e_ident[EI_MAG2]
    ELFMAG3     'F'     e_ident[EI_MAG3]

文件使用的数据类型： ::

    ELFCLASSNONE    0   非法数据类型
    ELFCLASS32      1   使用32位机器数据类型
    ELFCLASS64      2   使用64位机器数据类型

数据字节序定义如下，为了解析效率，字节序尽量与目标平台字节序保持一致： ::

    ELFDATANONE     0   非法字节序
    ELFDATA2LSB     1   二进制补码低字节位于低地址（小端字节序）
    ELFDATA2MSB     2   二进制补码高字节位于低地址（大端字节序）

使用的补充规范，处理器补充文档（psABI）可以定义 64~255 范围内的自己特定的值，如果 psABI
没有定义自己的值，则应该使用下面这些预定义的值，如果设置成 0 则表示没有使用任何扩展。如
果设置的值在 64~255 范围内，它的含义依赖于 ELF 头部信息中的 e_machine 的解释。 ::

    ELFOSABI_NONE       0 未指定或没有使用扩展
    ELFOSABI_HPUX       1 Hewlett-Packard HP-UX
    ELFOSABI_NETBSD     2 NetBSD
    ELFOSABI_GNU        3 GNU
    ELFOSABI_LINUX      3 Linux，它是 ELFOSABI_GNU 的别名（历史原因）
    ELFOSABI_SOLARIS    6 Sun Solaris
    ELFOSABI_AIX        7 AIX
    ELFOSABI_IRIX       8 IRIX
    ELFOSABI_FREEBSD    9 FreeBSD
    ELFOSABI_TRU64      10 Compaq TRU64 UNIX
    ELFOSABI_MODESTO    11 Novell Modesto
    ELFOSABI_OPENBSD    12 Open BSD
    ELFOSABI_OPENVMS    13 Open VMS
    ELFOSABI_NSK        14 Hewlett-Packard Non-Stop Kernel
    ELFOSABI_AROS       15 Amiga Research OS
    ELFOSABI_FENIXOS    16 The FenixOS highly scalable multi-core OS
    ELFOSABI_CLOUDABI   17 Nuxi CloudABI
    ELFOSABI_OPENVOS    18 Stratus Technologies OpenVOS
                        64-255 平台特定值范围（Architecture-specific value range）

程序分区
---------

目标文件中的每个分区都对应有唯一一个分区头部，可以存在一个分区头部而没有对应的分区。每个
分区占据一块连续的字节空间（可能为空），目标文件中的分区不能相互覆盖，即一个字节数据不能
同时属于多个分区；目标文件中可以有非活动空间，非活动空间中的数据是未指定的。

分区头部表由分区头部结构体构成： ::

    typedef struct {
        Elf32_Word sh_name;
        Elf32_Word sh_type;
        Elf32_Word sh_flags;
        Elf32_Addr sh_addr;
        Elf32_Off sh_offset;
        Elf32_Word sh_size;
        Elf32_Word sh_link;
        Elf32_Word sh_info;
        Elf32_Word sh_addralign;
        Elf32_Word sh_entsize;
    } Elf32_Shdr;

    typedef struct {
        Elf64_Word sh_name;
        Elf64_Word sh_type;
        Elf64_Xword sh_flags;
        Elf64_Addr sh_addr;
        Elf64_Off sh_offset;
        Elf64_Xword sh_size;
        Elf64_Word sh_link;
        Elf64_Word sh_info;
        Elf64_Xword sh_addralign;
        Elf64_Xword sh_entsize;
    } Elf64_Shdr;

sh_name
    分区名称，字符串分区的索引，名称是以NUL字符结束的字符串
sh_type
    分区类型，不同类型的分区的内容和语义不同
sh_flags
    分区属性标记
sh_addr
    如果分区会出现在进程的内存映像中，该地址指定分区第一个字节的地址，否则为 0
sh_offset
    分区在文件中的偏移字节，如果分区的类型是 SHT_NOBITS，它不占用文件空间，该值是概念
    上的位置
sh_size
    分区大小，如果是 SHT_NOBITS 分区，该值可以不是 0，但是分区不占用文件空间
sh_link
    sh_link 的解释依赖于分区的类型，通常是关联的分区头部索引
sh_info
    分区的额外信息，含义依赖于分区的类型
sh_addralign
    有些分区有地址对齐的要求，当前 0 和 1 表示没有对齐要求，2 和 2 的幂表示有对齐要求
sh_entsize
    有些分区包括多个固定大小的条目，例如符号表，该成员指定条目的大小，该值如果是 0 表示
    分区不包含固定大小的条目

分区头部通过索引值访问，存在一些特殊的分区索引值如下。预留在 SHN_LORESERVE 和 SHN_HIRESERVE
之间的索引值不是一个真实的头部索引，分区头部表中不包含这种索引的分区。 ::

    SHN_UNDEF       0       第一个分区头部或未定义分区
    SHN_LORESERVE   0xff00  低位预留
    SHN_LOPROC      0xff00  处理器预留
    SHN_HIPROC      0xff1f  处理器预留
    SHN_LOOS        0xff20  操作系统预留
    SHN_HIOS        0xff3f  操作系统预留
    SHN_ABS         0xfff1  指定该索引的符号的值是不受重定位影响的绝对值
    SHN_COMMON      0xfff2  指定该索引的符号是一个通用符号，例如 Fortran 的 COMMON 或未分配的 C 外部变量
    SHN_XINDEX      0xffff  占位值
    SHN_HIRESERVE   0xffff  高位预留

其中的第一个分区头部是一个特殊的头部，它包含以下信息： ::

    sh_name         0           没有名称
    sh_type         SHT_NULL    非活动分区头部
    sh_flags        0           没有分区属性
    sh_addr         0           没有分区地址
    sh_offset       0           没有对应的分区
    sh_size         0或头部个数  如果不是0表示实际的分区头部个数
    sh_link         0或分区索引  如果不是0表示分区名字字符串表头部的索引
    sh_info         0           没有额外信息
    sh_addralign    0           不需要对齐
    sh_entsize      0           分区不包含固定大小的条目

分区的类型（sh_type）： ::

    SHT_NULL            0
    SHT_PROGBITS        1
    SHT_SYMTAB          2
    SHT_STRTAB          3
    SHT_RELA            4
    SHT_HASH            5
    SHT_DYNAMIC         6
    SHT_NOTE            7
    SHT_NOBITS          8
    SHT_REL             9
    SHT_SHLIB           10
    SHT_DYNSYM          11
    SHT_INIT_ARRAY      14
    SHT_FINI_ARRAY      15
    SHT_PREINIT_ARRAY   16
    SHT_GROUP           17
    SHT_SYMTAB_SHNDX    18
    SHT_LOOS            0x60000000
    SHT_HIOS            0x6fffffff
    SHT_LOPROC          0x70000000
    SHT_HIPROC          0x7fffffff
    SHT_LOUSER          0x80000000
    SHT_HIUSER          0xffffffff

SHT_NULL
    非活动分区头部，没有关联实际分区
SHT_PROGBITS
    程序分区，包含程序定义的信息
SHT_SYMTAB
    符号表分区，当前每个文件只能包含一个该分区
SHT_STRTAB
    字符串分区，每个文件可以包含多个字符串分区
SHT_RELA 
    重定位分区，每个文件可以包含多个重定位分区，重定位分区由重定位条目组成
SHT_REL
    重定位分区，包含重定位条目，但是条目内容不包含显式的附加值，每个文件可以包含多个重
    定位分区
SHT_HASH
    哈希表分区，包含符号哈希表，当前每个文件只能包含一个哈希表分区
SHT_DYNAMIC
    动态链接分区，包含动态链接信息，只能包含一个动态链接分区
SHT_NOTE
    说明分区，包含文件的说明注释
SHT_NOBITS
    该分区不实际占用文件空间，但类似于程序分区；可以拥有非零的分区偏移值，表示概念上的
    文件偏移
SHT_SHLIB
    预留分区，当前没有指定具体含义
SHT_DYNSYM
    符号表分区，当前每个文件只能包含一个该分区，仅包含最小动态链接符号集
SHT_INIT_ARRAY
    该分区包含初始函数数组，每个函数不带参数也没有返回值
SHT_FINI_ARRAY
    该分区包含终止函数数组，每个函数不带参数也没有返回值
SHT_PREINIT_ARRAY
    该分区包含的数组中的函数，会在所有初始函数执行前执行，每个函数不带参数也没有返回值
SHT_GROUP
    该分区定义一个分区组合，分区组合是一组关联的可以让链接器特别对待的分区，只能存在于可
    重定位目标文件中，该分区头部必须位于所有包含的分区头部的前面
SHT_SYMTAB_SHNDX
    该分区与一个符号表分区关联，符号表中如果包含值为 SHN_XINDEX 的分区头部索引，该分区
    包含对应符号引用地实际索引或 0
SHT_LOOS SHT_HIOS
    操作系统特殊语义预留
SHT_LOPROC SHT_HIPROC
    处理器特殊语义预留
SHT_LOUSER SHT_HIUSER
    应用程序预留，应用程序可以自由使用这些值而不用担心和系统定义值冲突

分区属性标记（sh_flags）： ::

    SHF_WRITE               0x01
    SHF_ALLOC               0x02
    SHF_EXECINSTR           0x04
    SHF_MERGE               0x10
    SHF_STRINGS             0x20
    SHF_INFO_LINK           0x40
    SHF_LINK_ORDER          0x80
    SHF_OS_NONCONFORMING    0x100
    SHF_GROUP               0x200
    SHF_TLS                 0x400
    SHF_COMPRESSED          0x800
    SHF_MASKOS              0x0ff00000
    SHF_MASKPROC            0xf0000000

SHF_WRITE
    分区包含的数据是可写的
SHF_ALLOC
    分区在进程执行时会占用内存，如果分区不存在于目标文件的内存映像中，会关闭该属性
SHF_EXECINSTR
    分区包含可执行的机器指令
SHF_MERGE
    该分区中的数据可能进行合并来减少重复，除非设置了 SHF_STRINGS 否则该分区包含的都是
    相同大小的条目。该分区中的每个元素都会与其他分区中相同名称（sh_name）、相同类型
    （sh_type）、相同属性（sh_flags）的元素进行比较。拥有相同值的元素会在程序运行时进
    行合并。如果两个或多个元素在程序运行时具有相同的值，它们会被合并为一个元素，而所有
    指向这些元素的重定位引用都需要更新，指向合并后的位置。为了确保合并操作的正确性，必
    须对所有可重定位的值进行分析，包括那些在运行时会导致重定位的值，来确定它们在运行时
    是否真的相同。遵从 ABI 规范的目标文件不能依赖于被合并的特定元素，遵从 ABI 规范的链
    接编辑器也可能选择不合并某些特定的元素。
SHF_STRINGS
    分区包含NUL结尾的字符串，每个字符的大小由分区头部中的 sh_entsize 指定
SHF_INFO_LINK
    分区头部中的 sh_info 成员的值是一个分区头部索引
SHF_LINK_ORDER
    这个标记添加特别的链接顺序，典型用法是构建一个按地址顺序引用代码分区或数据分区的表。
    如果分区头部字段 sh_link 引用了另一个分区，这个分区需要按照链接到的分区的相对位置
    顺序来排列。
SHF_OS_NONCONFORMING
    这种分区需要标准链接规则之外的特殊操作系统规则来保证正确的链接行为。如果当前分区的
    sh_type 字段或者 sh_flags 字段包含操作系统预留范围内的值，而且处理该分区的链接编辑
    器不识别这些值，链接器应该报错拒绝包含这个分区的目标文件。
SHF_GROUP
    此分区是分区组合的一个成员分区，必须被一个 SHT_GROUP 分区引用；只有重定位目标文件
    中的分区才能设置为成员分区
SHF_TLS
    该分区包含线程本地存储
SHF_COMPRESSED
    该分区包含压缩的数据，不能和 SHF_ALLOC 一起使用，也不能应用到 SHT_NOBITS 分区
SHF_MASKOS SHF_MASKPROC
    处理器特殊语义预留

压缩分区以压缩头部结构体开始，对于一个压缩分区的所有重定位都要指定一个非压缩分区数据的偏
移，因此在重定位之前，必须先解压分区数据。每个压缩分区可以独立指定自己的压缩算法，一个目
标文件中的不同分区可以使用不同的压缩算法。 ::

    typedef struct {
        Elf32_Word ch_type;
        Elf32_Word ch_size;
        Elf32_Word ch_addralign;
    } Elf32_Chdr;

    typedef struct {
        Elf64_Word ch_type;
        Elf64_Word ch_reserved;
        Elf64_Xword ch_size;
        Elf64_Xword ch_addralign;
    } Elf64_Chdr;

ch_type
    使用的压缩算法类型
ch_size
    指定原未压缩数据的字节数
ch_addralign
    原未压缩数据的地址对齐要求

压缩算法类型，其中 ZLIB 算法（http://zlib.net）的压缩的数据紧随压缩头部结构体之后： ::

    ELFCOMPRESS_ZLIB    1
    ELFCOMPRESS_LOOS    0x60000000
    ELFCOMPRESS_HIOS    0x6fffffff
    ELFCOMPRESS_LOPROC  0x70000000
    ELFCOMPRESS_HIPROC  0x7fffffff

依赖于分区类型的字段 sh_link 和字段 sh_info 的值： ::

    分区类型            sh_link                             sh_info
    SHT_DYNAMIC         分区中的条目使用的字符串表分区的索引    0
    SHT_HASH            哈希表关联的符号表分区的索引           0
    SHT_REL/RELA        关联的符号表分区的索引                应用重定位的分区的索引
    SHT_SYMTAB/DYNSYM   关联的字符串表分区的索引              第一个非本地符号的索引
    SHT_GROUP           关联的符号表分区的索引                符号索引，该符号提供了组合分区的签名
    SHT_SYMTAB_SHNDX    关联的符号表分区的索引                0

分区链接规则
-------------

链接编辑器在处理分区头部时如果遇到不识别的操作系统预留范围内的值（例如字段 sh_type 或者
sh_flags），需要按以下规则处理：

1. 如果分区 sh_flags 字段包含了 SHF_OS_NONCONFORMING 标记，意味着该分区需要特殊的操
   作系统规则才能被正确处理。这种情况下，链接器应该拒绝包含该分区的对象文件，并报错；
2. 不识别的未包含 SHF_OS_NONCONFORMING 标记的分区，链接器通过两个阶段的过程来合并这些
   分区，这些分区需要对齐到规定的对齐地址上，合并后的分区也需要对齐到这些分区中的最大对
   齐地址上；
3. 合并不识别分区的第一步，如果分区的名称、类型和属性标志匹配，应该串联成一个单独的分区，
   串联顺序应该满足任何已知输入分区的属性要求（例如 SHF_MERGE 和 SHF_LINK_ORDER），如
   果没有约束，分区应该按照输入顺序合并；
4. 合并不识别分区的第二步，根据分区的属性标志，分区应该分配到对应的分段或单元。除非有不
   兼容的属性，每种特定的不识别类型应该分配到同一个单元中。在单元内部，相同的不识别类型
   分区应该放到一起；
5. 非操作系统特定的处理，例如重定向，应该同样作用域不识别分区。输出目标文件如果存在分区
   头部表，应该包含不识别分区的分区头部，并且移除任何不识别的属性标记；

推荐链接编辑器也使用上面的两步规则来处理已知类型的分区。这些分区之间的填补空间，只要合适
也可以包含非零值。

分区分组
---------

目标文件中的一些分区可能是内部相关的一组分区中的一员。例如，内联函数的非内联定义可能不仅
需要包含其可执行指令的分区，还需要一个只读数据分区来包含被引用的字面常量，以及一个或多个
调试信息分区和其他信息分区等等。而且，这些分区之间可能存在内部引用，如果其中的某一分区被
移除或被另一个目标文件中的副本替换，这些引用将没有意义。因此这样的一组分区必须作为一个整
体被包含或被移除。另外，一个分区不能同时属于多个分区组合。

分区类型 SHT_GOURP 用来定义这种分区组合，该分区头部中的 sh_link 和 sh_info 用于指定该
分区关联的符号，该符号用于组合分区的签名。其中 sh_link 指定符号所属符号表，sh_info 指
定符号在符号表中的索引。组合分区的分区属性为 0，分区名称也是 0，关联的符号所在符号表分区
不需要是该分区组合的成员。

组合分区包含的数据是一个 Elf32Word 类型的数组，第一个 Elf32Word 是一个标记，剩下的是成
员分区的分区头部索引，成员分区必须设置 SHF_GOURP 分区属性。组合分区的分区标记如下： ::

    GRP_COMDAT      0x1
    GRP_MASKOS      0x0ff00000
    GRP_MASKPROC    0xf0000000

GRP_COMDAT
    这是一个 COMDAT 组合分区，这种分区中的内容可能与其他目标文件中的 COMDAT 组合分区重
    复，是否重复由组合分区的签名（sh_info）定义。这种情况下，重复的组合分区只有一个会被
    链接器保留，剩余的组合分区的所有成员分区都会被移除
GRP_MASKOS
    操作系统预留
GRP_MASKPROC
    处理器预留

当链接器移除一个组合分区时，必须移除该组内的所有成员分区，这是为了维护程序的一致性和正确
性，防止因为部分移除而导致其他分区出现出现不可预期的行为。但这不意味着在移除调试信息等特
殊分区时，必须移除调试信息所引用的分区，即使这些分区也是组合分区的一部分。也即，链接器在
处理特殊类型的分区时，可能需要采取不同的策略，而不是简单地移除整个分组内的所有分区。

当移除分区组合时，为了避免引用悬置，以及对符号表进行最少处理，需要遵循以下规则：

1. 如果一个符号表中有 STB_GLOBAL 或 STB_WEAK 绑定属性的符号，且这个符号定义在组合分区
   中，同时这个符号表分区不属于这个组合分区，那么当组合分区被移除时这些符号需要转换成未
   定义符号（SHN_UNDEF）
2. 如果一个符号表中有 STB_LOCAL 绑定属性的符号，且这个符号定义在组合分区中，同时这个符
   号表不属于这个组合分区，那么当组合分区被移除时这个符号也必须移除
3. 如果一个未定义的符号只被组合分区引用，并且这个符号所在的符号表分区不属于这个组合分区，
   当组合分区被移除时这个未定义符号不会被移除
4. 在组合分区外部，不允许引用组合分区里的非符号，例如不允许在 sh_link 或 sh_info 字段
   中使用组合分区的头部索引

特殊分区
---------

存在一些特殊的分区，有固定的分区类型和属性。以点号开始的分区名称是系统预留的名称，一个目
标文件可以有多个相同名称的分区。处理器架构相关的分区以 e_machine 字段定义的处理器名称开
头例如 .AARCH64.procsec。 ::

    分区名称         分区类型            分区属性
    .bss             SHT_NOBITS         SHF_ALLOC|WRITE
    .comment         SHT_PROGBITS       0
    .data            SHT_PROGBITS       SHF_ALLOC|WRITE
    .data1           SHT_PROGBITS       SHF_ALLOC|WRITE
    .debug           SHT_PROGBITS       0
    .dynamic         SHT_DYNAMIC        SHF_ALLOC|O(WRITE)
    .dynstr          SHT_STRTAB         SHF_ALLOC
    .dynsym          SHT_DYNSYM         SHF_ALLOC
    .fini            SHT_PROGBITS       SHF_ALLOC|EXECINSTR
    .fini_array      SHT_FINI_ARRAY     SHF_ALLOC|WRITE
    .got             SHT_PROGBITS       见处理器补充规范
    .hash            SHT_HASH           SHF_ALLOC
    .init            SHT_PROGBITS       SHF_ALLOC|EXECINSTR
    .init_array      SHT_INIT_ARRAY     SHF_ALLOC|WRITE
    .interp          SHT_PROGBITS       O(SHF_ALLOC)
    .line            SHT_PROGBITS       0
    .note            SHT_NOTE           0
    .plt             SHT_PROGBITS       见处理器补充规范
    .preinit_array   SHT_PREINIT_ARRAY  SHF_ALLOC|WRITE
    .relNAME         SHT_REL            O(SHF_ALLOC)
    .relaNAME        SHT_RELA           O(SHF_ALLOC)
    .rodata          SHT_PROGBITS       SHF_ALLOC
    .rodata1         SHT_PROGBITS       SHF_ALLOC
    .shstrtab        SHT_STRTAB         0
    .strtab          SHT_STRTAB         O(SHF_ALLOC)
    .symtab          SHT_SYMTAB         O(SHF_ALLOC)
    .symtab_shndx    SHT_SYMTAB_SHNDX   O(SHF_ALLOC)
    .tbss            SHT_NOBITS         SHF_ALLOC|WRITE|TLS
    .tdata           SHT_PROGBITS       SHF_ALLOC|WRITE|TLS
    .tdata1          SHT_PROGBITS       SHF_ALLOC|WRITE|TLS
    .text            SHT_PROGBITS       SHF_ALLOC|EXECINSTR

.bss
    未初始化数据，程序开始运行时初始化为全零，该分区不占据文件空间
.comment
    该分区包含版本控制信息
.data .data1
    初始化数据
.debug
    符号调试信息，并且所有以 .debug 开始的名称都预留给以后使用
.dynamic
    动态链接信息，是否可写由特定处理器决定
.dynstr
    动态链接对应的字符串表
.dynsym
    动态链接符号表
.fini
    进程终止时执行的指令
.fini_array
    函数指针数组，可执行或动态目标文件在进程终止时需要执行的函数
.got
    全局偏移表
.hash
    符号哈希表
.init
    进程启动后调用主函数之前执行的指令
.init_array
    可执行或动态目标文件在初始化阶段需要执行的函数
.interp
    包含程序解释器的路径名称
.line
    符号调试用的行号信息，描述源程序和机器码之间的行号关系
.note
    文件说明信息
.plt
    过程链接表
.preinit_array
    可执行或动态目标文件在初始化阶段之前需要执行的函数
.relNAME .relaNAME
    重定位信息，如果文件包含一个包含重定位分区的可加载分段，分区需要包含 ALLOC 属性否则
    为 0。按照惯例，NAME 表示的是重定位需要应用的分区，例如 .text 对应的重定位分区为
    .rel.text 或 .rela.text
.rodata .rodata1
    只读数据
.shstrtab
    分区名称字符串
.strtab
    字符串表，如果文件的一个可加载分段包含这个字符串表则需要指定 ALLOC 属性否则为 0
.symtab
    符号表，如果文件的一个可加载分段包含这个符号表则需要指定 ALLOC 属性否则为 0
.symtab_shndx
    符号表分区头部索引，如果对应的符号表有 ALLOC 属性那么这个分区也需要指定 ALLOC 否则
    为 0
.tbss
    未初始化TLS数据
.tdata .tdata1
    初始化TLS数据
.text
    程序可执行指令
.conflict .gptab .liblist .lit4 .lit8 .reginfo .sbss .sdata .tdesc
    历史原因遗留下来的一些分区名称

字符串表分区
-------------

字符串表分区包含以NUL结束的字符串，目标文件使用这些字符串来表示符号或者分区的名称。分区
的第一个字节，字符串索引0，是一个NUL字符，表示一个空字符串；因此索引为0的字符串表示一个
没有名称或者空名称。空字符串分区是允许的，此时分区头部中的 size 字段的值为零，对于空分
区非0索引是非法的。

符号表分区
-----------

符号表分区由符号结构体组成，包含了程序符号定义和引用的定位和重定位信息。分区中的第一个符
号是未定义符号，对应的索引是 STN_UNDEF（值为零）。这个特殊的第一个符号的内容为： ::

    st_name     0           没有名称
    st_value    0           零值
    st_size     0           没有大小
    st_info     0           没有类型，绑定属性为本地符号
    st_other    0           默认可见性
    st_shndx    SHN_UNDEF   无关联分区，未定义的符号

符号的结构体定义如下： ::

    typedef struct {
        Elf32_Word st_name;
        Elf32_Addr st_value;
        Elf32_Word st_size;
        Elf32_Byte st_info;
        Elf32_Byte st_other;
        Elf32_Half st_shndx;
    } Elf32_Sym;

    typedef struct {
        Elf64_Word st_name;
        Elf64_Byte st_info;
        Elf64_Byte st_other;
        Elf64_Half st_shndx;
        Elf64_Addr st_value;
        Elf64_Xword st_size;
    } Elf64_Sym;

st_name
    符号名称的字符串索引，索引0表示符号没有名称，字符串表由分区头部 sh_link 字段定义
st_value
    符号的值，根据上下文该值可能是一个绝对值、一个地址等等；如果符号的值是一个分区里的一
    个位置，那么 shndx 指定该分区头部索引，当这个分区因为重定位移动了，该符号的值也要随
    之改变；在重定位文件中，符号的值是定义该符号的分区的偏移（文件表示）；在可执行或共享
    目标文件中，符号的值是一个虚拟地址（内存表示），方便符号的动态链接
st_size
    符号关联对象的大小，例如符号表示一个 C 语言类型，该值是该类型的大小
st_info
    符号的类型和绑定属性，低 4-bit 是类型信息，高 4-bit 是绑定属性
st_other
    当前该字段表示符号的可见性，仅低 2-bit 有效
st_shndx
    当符号值是分区的位置时需要关联该分区，该字段表示这个关联分区的分区头部索引，如果这个
    索引值大于等于 SHN_XINDEX（0xffff），该字段设为 SHN_XINDEX，具体索引值由 SHT_SYMTAB_SHNDX
    类型的分区定义，是哪个 SHT_SYMTAB_SHNDX 分区需要查找分区头部表，检查 SHT_SYMTAB_SHNDX
    类型的分区，其 sh_link 字段是否指向该符号分区的头部索引，并检查 SHT_SYMTAB_SHNDX
    分区中对应符号位置的索引。如果该字段是 SHN_ABS 表示该符号的值是一个绝对值，不会被重
    定位影响；如果是 SHN_UNDEF 表示该符号是未定义符号，当链接器将当前文件与另一个定义了
    该符号的文件一起链接时，该符号会链接到这个具体定义；如果是 SHN_COMMON 表示该符号标
    记的是一个还未被分配的未初始化通用块，符号的值是地址对齐要求，链接器会在对应地址对齐
    的位置分配这个符号的存储空间，符号的 st_size 字段表示该通用块的大小，该符号只能出现
    在重定位文件中

符号的类型和绑定属性定义如下： ::

    #define ELF32_ST_BIND(i)   ((i)>>4)
    #define ELF32_ST_TYPE(i)   ((i)&0xf)
    #define ELF32_ST_INFO(b,t) (((b)<<4)+((t)&0xf))

    #define ELF64_ST_BIND(i)   ((i)>>4)
    #define ELF64_ST_TYPE(i)   ((i)&0xf)
    #define ELF64_ST_INFO(b,t) (((b)<<4)+((t)&0xf))

    STB_LOCAL   0
    STB_GLOBAL  1
    STB_WEAK    2
    STB_LOOS    10
    STB_HIOS    12
    STB_LOPROC  13
    STB_HIPROC  15

    STT_NOTYPE  0
    STT_OBJECT  1
    STT_FUNC    2
    STT_SECTION 3
    STT_FILE    4
    STT_COMMON  5
    STT_TLS     6
    STT_LOOS    10
    STT_HIOS    12
    STT_LOPROC  13
    STT_HIPROC  15

符号的绑定属性：

STB_LOCAL
    本地符号，在目标文件外部不可见，在链接时本地符号优先于全局符号和弱符号，在符号表的分
    区头部结构体中，sh_info 字段保存了该符号表的第一个非本地符号的索引
STB_GLOBAL
    全局符号，在所有目标文件中可见，一个目标中定义的全局符号可以解决另一个目标文件对相同
    符号的引用
STB_WEAK
    类似于全局符号，但是它们的定义优先级低；弱符号通常用于系统软件，在应用程序中使用弱符
    号是不可靠的，因为运行时环境的改变可能导致执行失败
STB_LOOS STB_HIOS
    操作系统语义预留
STB_LOPROC STB_HIPROC
    操作系统语义预留

符号的绑定属性定义了链接可见性和行为，其中全局符号和弱符号有两个主要不同：

1. 全局符号不允许多个定义，但是弱符号可以有一个同名的全局符号定义，并使用全局符号而忽略
   弱符号；相同的，如果符号是一个通用符号（st_shndx 值为 SHN_COMMON），链接器也会选择
   该通用符号而忽略弱符号
2. 当链接器搜索归档库文件时，它会解压未定义全局符号在其中找到的包含定义的成员文件，该定
   义可以是一个全局符号或者弱符号；但是链接器不会解压归档成员文件来解决未定义的弱符号，
   未解决的弱符号的值是零

符号的类型：

STT_NOTYPE
    符号类型未指定
STT_OBJECT
    符号是一个数据对象，例如变量、数组等等
STT_FUNC
    符号是一个函数或其他可执行代码
STT_SECTION
    符号关联的是一个分区，该类型符号主要用于重定位并拥有 STB_LOCAL 绑定属性
STT_FILE
    通常，该符号名称指定的是目标文件的源文件名称。文件符号的绑定属性为 STB_LOCAL，它关
    联的分区是 SHN_ABS（表示该符号的值是一个不受重定位影响的绝对值）。如果这个符号存在，
    必须出现在所有本地符号之前
STT_COMMON
    该符号表示一个未初始化的公共块
STT_TLS
    该符号表示一个 TLS 条目，该符号的值是一个给定的文件偏移而不是实际的地址。该类型的符
    号只能被特殊的 TLS 重定位条目引用，并且 TLS 重定位条目只能引用 STT_TLS 类型的符号
STT_LOOS STT_HIOS
    操作系统语义预留
STT_LOPROC STT_HIPROC
    处理器语义预留

共享目标文件中的函数符号有特殊重要程度，当另一个目标文件引用共享文件中的函数时，链接编辑
器会自动给这个函数符号的引用创建一个过程链接表条目。而对共享对象中其他类型符号，不会自动
通过过程链接表的方式来引用。

类型为 STT_COMMON 的符号表示一个未初始化通用块。在重定位文件中，该类型的符号是未分配的，
必须关联一个特殊的分区索引 SHN_COMMON，表示是一个还未被分配的未初始化通用块。在可执行文
件以及共享目标文件中，该类型的符号必须已经在定义的相同分区中进行了分配。

在重定位目标中，STT_COMMON 类型的符号被当成跟设定了 SHN_COMMON 的其他类型的符号一样对
待。如果链接编辑器为 SHN_COMMON 符号在输出目标文件中分配了空间，必须将输出的符号类型设
为 STT_COMMON。

当动态链接器遇到一个符号引用被解析到一个定义的 STT_COMMON 类型符号时，它可以（不必须）
将符号的解析规则修改为：

1. 不讲符号绑定到找到的第一个符号，而是绑定到第一个非 STT_COMMON 类型的符号；
2. 如果没有这种符号，则查找同名 STT_COMMON 类型的定义中大小最大的那个进行绑定；

符号的可见性定义如下，重定位文件中的隐藏符号和内部使用符号包含到可执行或共享目标文件中时
要么移除要么转成成本地符号。文件内部引用的保护符号，一定会解析为该文件中定义的这个保护符
号，即使这个保护符号是弱符号。本地符号不能使用保护可见性，如果共享目标文件中定义的保护符
号被用来解决另一个可执行或共享目标文件的引用，对应的未定义符号会被设成默认可见性。 ::

    #define ELF32_ST_VISIBILITY(o) ((o)&0x3)
    #define ELF64_ST_VISIBILITY(o) ((o)&0x3)

    STV_DEFAULT     0
    STV_INTERNAL    1
    STV_HIDDEN      2
    STV_PROTECTED   3

STV_DEFAULT
    默认可见性，即绑定属性定义的链接可见性，本地符号是隐藏的，全局和弱符号是全局可见的且
    具抢占性
STV_INTERNAL
    内部可见，由处理器补充规范定义的更强隐藏性，补充规范的定义应该允许将内部可见符号安全
    的当成隐藏符号
STV_HIDDEN
    符号是隐藏的，表示符号对外部不可见并且是保护的，注意由该符号命名的对象如果它的地址传
    递到了外部仍然可以被外部模块引用
STV_PROTECTED
    保护的符号对外部可见，但不会被外部定义的相同符号抢占，即使该符号的默认规则可以被外部
    符号覆盖

符号的可见性不会影响链接编辑器对可执行文件或共享目标文件中的未定义符号的解析，这种解析基
于符号的绑定属性。当符号解析完之后，才会应用符号的可见性：

1. 首先，当非默认可见性应用到符号引用时，表示的是这个符号引用必须是可执行文件或共享文件
   内部定义的符号，如果没有定义，这个引用必须是一个弱绑定（STB_WEAK）并被解析的值为零；
2. 第二，如果任意一个符号引用或符号定义的可见性不是默认可见性，对应的可见性必须应用到链
   接后的目标文件中，如果同一个符号引用或定义具有不同的可见性，使用最强约束的那一个，约
   束性从强到弱一次为 STV_INTERNAL、STV_HIDDEN、STV_PROTECTED；

重定位分区
-----------

重定位用于处理符号引用及其定义的联系，例如当程序调用一个函数时，对应的调用指令必须将代码
控制权转移到合适的目标地址。重定位文件必须有重定位条目来描述怎样修改它的分区内容，从而允
许可执行或共享目标文件为程序的进程映像创建正确的内容。

重定位分区由重定位结构体组成： ::

    typedef struct {
        Elf32_Addr r_offset;
        Elf32_Word r_info;
    } Elf32_Rel;

    typedef struct {
        Elf32_Addr r_offset;
        Elf32_Word r_info;
        Elf32_Sword r_addend;
    } Elf32_Rela;

    typedef struct {
        Elf64_Addr r_offset;
        Elf64_Xword r_info;
    } Elf64_Rel;

    typedef struct {
        Elf64_Addr r_offset;
        Elf64_Xword r_info;
        Elf64_Sxword r_addend;
    } Elf64_Rela;

r_offset
    重定位应用的位置，重定位文件这个值是分区开始的偏移，可执行或共享目标文件是受重定位
    影响的存储单元的虚拟地址；具体应用到哪个分区，由重定位分区头部字段 sh_info 指定
r_info
    指定需要重定位的符号索引（高 24 位或高 32 位）以及重定位类型（低 8 位或低 32 位），
    符号是哪个符号表中的符号由重定位分区头部字段 sh_link 指定；例如，一个调用指令的重定
    位条目需要指定被调函数的符号索引，如果使用未定义的符号索引 STN_UNDEF，重定位使用 0
    作为符号值；重定位的类型是处理器相关的，它的描述在处理器补充规范中
r_addend
    计算可重定位字段值需要显式附加的值，重定位条目类型 ElfRel 在需要修改的位置隐式保存
    了需要附加的值；根据具体处理器，这两种方式可能某一种会更方便，因此特定机器的实现可
    能根据具体上下文使用其中一种方式

重定位信息（r_info）中的符号索引和重定位类型，重定位类型由处理器补充规范定义： ::

    #define ELF32_R_SYM(i)    ((i)>>8)
    #define ELF32_R_TYPE(i)   ((Elf32_Byte)(i))
    #define ELF32_R_INFO(s,t) (((s)<<8)+(Elf32_Byte)(t))

    #define ELF64_R_SYM(i)    ((i)>>32)
    #define ELF64_R_TYPE(i)   ((i)&0xffffffffL)
    #define ELF64_R_INFO(s,t) (((s)<<32)+((t)&0xffffffffL))

重定位分区关联了两个其他分区，一个是符号表分区（由重定位分区头部字段 sh_link 指定），一
个是修改的分区（由头部字段 sh_info 指定）。重定位典型应用的步骤是：

1. 确定引用的符号值
2. 提取附加数
3. 将重定位类型所隐含的表达式应用于符号和加数
4. 提取表达式结果的所需部分
5. 将这部分结果保存到需要重定位的字段中

如果多个连续的重定位条目应用于同一个 offset 指定的位置，它们不单独应用，而是组合应用，
组合应用意味着上述标准应用方式需要修改为：

1. 除了组合中最后一个重定位操作，其他重定位操作的表达式结果以适用的处理器规范定义的完整
   指针的精度保留，而不是提取部分结果并保存到重定位字段
2. 除了组合中的第一个重定位操作，其他重定位操作使用的附加数是前一次重定位操作保留的表达
   式结果
3. 处理器补充规范可能会指定单独的重定位类型，总是阻止组合应用，或总是开启一个新的重定位
   应用

重定位类型
-----------

由处理器补充规范定义。

程序加载
=========

这部分描述目标文件信息，以及创建运行程序的系统行为。目标文件的一些信息是在所有系统上通用
的，还有一些则依赖于特定平台。可执行文件或共享目标文件表示的是一个静态的程序，为了执行这
个程序，系统需要使用这个文件，并创建一个动态程序或进程映像。一个进程映像包含了多个分段，
例如代码、数据、栈等等。

程序头部
---------

可执行文件或共享目标文件的程序头部表，由程序头部结构体构成，每个程序头部描述一个程序分段
或者系统需要的用于准备执行程序的其他信息。一个程序分段（Segment）包含一个或多个分区
（Section）。程序头部只对可执行文件或共享目标文件有意义，程序头部的大小以及程序头部的个
数由文件头部字段 e_phentsize 和 e_phnum 指定。程序头部结构体定义如下： ::

    typedef struct {
        Elf32_Word p_type;
        Elf32_Off p_offset;
        Elf32_Addr p_vaddr;
        Elf32_Addr p_paddr;
        Elf32_Word p_filesz;
        Elf32_Word p_memsz;
        Elf32_Word p_flags;
        Elf32_Word p_align;
    } Elf32_Phdr;

    typedef struct {
        Elf64_Word p_type;
        Elf64_Word p_flags;
        Elf64_Off p_offset;
        Elf64_Addr p_vaddr;
        Elf64_Addr p_paddr;
        Elf64_Xword p_filesz;
        Elf64_Xword p_memsz;
        Elf64_Xword p_align;
    } Elf64_Phdr;

p_type
    程序头部类型
p_flags
    分段权限标记
p_offset
    分段第一个字节的文件偏移
p_vaddr
    分段第一个字节在内存的虚拟地址
p_paddr
    在物理地址有意义的系统上该字段是分段的物理地址，但 System V 忽略应用程序的物理地址，
    因此该字段的内容未指定
p_filesz
    分段在文件中的大小，可能为 0，但必须大于等于 p_memsz
p_memsz
    分段在内存映像中的大小，可能为 0
p_align
    表示 p_vaddr 和 p_offset 的地址必须除以 p_align 之后余数相同，也即这两值是同余的；
    值0和1表示没有地址要求，2和2的幂表示对地址有要求；根据处理器补充规范，可加载的进程
    分段的 p_vaddr 和 p_offset 值必须基于内存页大小同余

分段的程序头部类型： ::

    PT_NULL     0
    PT_LOAD     1
    PT_DYNAMIC  2
    PT_INTERP   3
    PT_NOTE     4
    PT_SHLIB    5
    PT_PHDR     6
    PT_TLS      7
    PT_LOOS     0x60000000
    PT_HIOS     0x6fffffff
    PT_LOPROC   0x70000000
    PT_HIPROC   0x7fffffff

PT_NULL
    未使用的程序头部，程序头部的其他字段的值未定义
PT_LOAD
    程序头部描述一个可加载分段，文件中的分段会映射到内存分段中，如果内存大小 p_memsz 大
    于文件大小 p_filesz，额外字节是 0；可加载分段程序头部必须按虚拟地址 p_vaddr 从小到
    大排序
PT_DYNAMIC
    分段内容是一个  SHF_DYNAMIC 分区（.dynamic），包含动态链接信息
PT_INTERP
    分段内容包含程序解释器的位置和路径字符串的大小，只能出现一次，如果存在它必须在所有可
    加载程序头部之前
PT_NOTE
    分段内容是一个说明分区，包含辅助信息的位置和大小
PT_SHLIB
    预留但当前未指定语义
PT_PHDR
    程序头部表自己的位置和大小，只能出现一次并且只有当程序头部表是内存映像的一部分时才有
    意义，如果存在它必须在所有可加载程序头部之前
PT_TLS
    分段内容是 TLS 模板
PT_LOOS
    操作系统特殊语义预留
PT_HIOS
    操作系统特殊语义预留
PT_LOPROC
    处理器殊语义预留
PT_HIPROC
    处理器殊语义预留

基地址
-------

程序头部的虚拟地址可能不是程序内存映像真实的虚拟地址；其中可执行文件通常包含绝对代码，为
了使进程正确执行，分段必须位于用来创建可执行文件的虚拟地址中；而共享目标文件通常包含位置
无关代码，这让分段虚拟地址随着不同进程而变化避免非法执行行为。在一些平台上，当系统为单个
进程选择虚拟地址时，它维护每个分段与其他共享对象分段的相对位置。由于这些平台位置无关代码
使用分段间的相对地址，内存中的虚拟地址的偏移必须与文件中虚拟地址匹配，这个偏移对于一个给
定的进程是一个常量值，称为基地址，基地址的一个用途是在动态链接中重定位文件的内存映像。

一个可执行文件或共享目标文件的基地址，在支持这个概念的平台上，是在执行时根据三个值计算的：
虚拟内存加载地址，最大的内存页大小，程序可加载分段的最低虚拟地址。首先内存加载地址会关联
到可加载分段的最低虚拟地址并截断到最近的最大内存页的整数倍，而分段的虚拟地址也会截断到最
近的最大内存页的整数倍，最后基地址的值是两个截断地址间的差。

分段权限
---------

被系统加载的程序必须有至少一个可加载分段，当系统创建可加载分段的内存映像时，根据程序头部
的 p_flags 字段来确定分段的权限。 ::

    PF_X        0x1         可执行
    PF_W        0x2         可写
    PF_R        0x4         可读
    PF_MASKOS   0x0ff00000  未指定
    PF_MASKPROC 0xf0000000  未指定


实际的内存权限由内存管理单元决定，系统可能授权比实际请求更多的权限，例如： ::

    请求权限        允许权限
    0x00            无权限
    PF_X            可读、可执行
    PF_W            可读、可写、可执行
    PF_W+PF_X       可读、可写、可执行
    PF_R            可读、可执行
    PF_R+PF_X       可读、可执行
    PF_R+PF_W       可读、可写、可执行
    PF_R+PF_W+PF_X  可读、可写、可执行

通常代码段是可读可执行的，但不可写；数据段是可读、可写、并且可执行的。

分段内容
---------

一个目标文件的分段由一个或多个分区组成，下面示意图仅表示分区内容常用的形式，分段中分区的
顺序以及分区间的关系具体可能不同，而且具体处理器补充规范还可能有自己的修改。

代码分段包含只读的指令和数据，通常包含以下分区： ::

    .text
    .rodata
    .hash
    .dynsym
    .dynstr
    .plt
    .rel.got

数据分段包含可写的数据和指令，通常包含以下分区： ::

    .data
    .dynamic
    .got
    .bss

一个 PT_DYNAMIC 类型的程序头部描述的是一个 .dynamic 分区，包含动态链接信息。而 .got
和 .plt 分区包含位置无关和动态链接信息。尽管 .plt 分区只出现在上图的代码分段中，但是根
据处理器的不同，可以存在于代码或数据分段中。具体见处理器补充规范的全局偏移表、过程链接表
部分。

根据前面的描述，.bss 分区是一个 SHT_NOBITS 类型不占用文件空间，但是它实际占用内存映像
的分段空间。通常，这些未初始化的数据位于分段的尾部，这一点可以通过将程序头部 p_memsz 字
段设置得比 p_filesz 字段更大来实现。

说明分段
---------

有时厂商或系统需要用特殊信息来标记目标文件，使得一些工具可以检查文件规范性和兼容性。分区
类型 SHT_NOTE 和分段程序头部类型 PT_NOTE 可以用作这个目的。说明分区包含可变数量的条目，
在 64 位目标文件格式中，每个条目是一个 8 字节字长的数组，而在32位目标文件格式中，每个条
目是 4 字节字长的数组。下图示意了说明分区的信息组织方式： ::

    namesz
    descsz
    type
    name ...
    desc ...

    包含两个条目的说明分区：

           | 0 | 1 | 2 | 3 |
    namesz |       7       |
    descsz |       0       |
    type   |       1       |
    name   | x | y | z |   |
           | c | o | \0|pad|
    namesz |       7       |
    descsz |       8       |
    type   |       3       |
    name   | x | y | z |   |
           | c | o | \0|pad|
    desc   | d | d | d | d |
           | d | d | d | d |

其中 name 是一个以 NUL 字符结束的字符串，namesz 是字符串大小（包括 NUL 字符），名字表
示所有者或发起者的名字，现在还没有正式的机制来规避名字冲突。如果没有名称，namesz 是 0，
但是 name 不能省略至少占一个机器字长。desc 包含描述信息，ABI 没有对描述内容设定规范，如
果没有描述内容，descsz 为 0。而 type 字段和 name 字段一起用来解释 desc 提供的描述信息
如何解析。

系统保留了 namesz 为 0，名称为空字符串（name[0] 是 NUL）的说明信息，所有其他名字必需至
少包含一个非 NUL 字符。

TLS 分段
---------

TLS 分段用来指定线程本地数据的大小和初始内容，对应的 PT_TLS 分段程序头部的信息如下：

p_offset
    TLS 初始映像的文件偏移位置
p_vaddr
    TLS 初始映像的内存虚拟地址
p_paddr
    保留
p_filesz
    TLS 初始映像的大小
p_memsz
    TLS 模板的总大小
p_flags
    PF_R 可读
p_align
    TLS 模板的地址同余要求

TLS 模板由所有标记是 SHF_TLS 的分区组成，存有初始化数据的 TLS 模板的位置是 TLS 初始映
像，TLS 模板的剩余部分由一个个 SHT_NOBITS 的 TLS 分区组成。

加载程序
---------

给定一个目标文件，系统必须将它加载到内存中才能运行其中的程序。具体定义见处理器补充规范。

动态链接
=========

当系统加载完程序后，还必须解决所有引用的符号来完成进程映像。

程序解释器
-----------

参与动态链接的可执行文件必须有一个 PT_INTERP 程序头部元素。在过程 exec(BA_OS) 执行过程
中，系统从该分段获取解释器路径，并从解释器文件分段创建出初始进程映像。也即，系统不使用原
本可执行文件的分段映像，而是为解释器创建一个内存映像。然后，是解释器的责任从系统接收控制
权并提供应用程序运行环境。

在进程初始化阶段，解释器接收系统控制权有两种方式，一种是获取到可执行文件定位在文件开头的
文件描述符，并使用该文件描述符来读取或映射可执行文件的分段到内存中；第二种是系统根据可执
行文件的格式，将可执行文件加载到内存种，而不是将一个打开的文件描述符交给解释器处理。除了
可能的文件描述符不同，解释器的初始进程状态与可执行文件本应接收到的状态是一致的。一个解释
器自身可能不需要获取第二个解释器。解释器可能是一个共享目标文件或者一个可执行文件。

共享目标文件会被地址无关的加载，最终的地址每个进程可能都不同，系统会使用 mmap(KE_OS) 过
程以及相关服务在动态链接分段区域创建出该共享目标的分段。因此，一个共享目标解释器不会与原
本的可执行文件的原始分段地址冲突。

而可执行文件可能被加载到一个固定地址，如果是这样，系统会根据程序头部中的虚拟地址创建分段。
因此，可执行文件解释器的虚拟地址可能与原本的可执行文件冲突，解释器需要负责解决这种冲突。

动态链接器
-----------

当创建一个使用动态链接的可执行文件时，链接器会添加一个 PT_INTERP 类型的程序头部到可执行
文件中，告诉系统调用动态链接器作为程序的解释器。系统提供的动态链接器的位置与特定处理器相
关。

过程 exec(BA_OS) 和动态链接器一起合作创建程序的进程映像，其步骤如下：

1. 将可执行文件的内存分段加载到进程映像
2. 将共享目标内存分段加载到进程映像
3. 执行可执行文件和它的共享目标的重定位
4. 如果将可执行文件的文件描述符传递给了动态链接器，关闭该文件描述符
5. 将控制权交给程序，就好像程序直接从 exec(BA_OS) 获得了控制权

链接编辑器也辅助动态链接器为可执行文件和共享目标文件创建可加载分段的数据，使得它们在执行
期间可用，具体的分段内容跟处理器相关。

1. 分段中的动态链接分区 .dynamic 包含其他动态链接信息的地址
2. 分段中的哈希分区 .hash 包含符号哈希表
3. 分段中的 .got 和 .plt 分区包含对应的全局偏移表和过程链接表

共享对象占据的虚拟内存地址可能跟文件中程序头部保存的地址不同。动态链接器重定位内存映像，
在应用程序获得控制权之前更新绝对地址。尽管只要将共享库加载到程序头部指定的地址，绝对地址
将是正确的，但是通常不会这样做。

如果进程的环境包含一个名为 LD_BIND_NOW 的非空值变量，动态链接器会在将控制权交给应用程序
之前处理所有的重定位。例如下面这些值都是非空值： ::

    LD_BIND_NOW=1
    LD_BIND_NOW=on
    LD_BIND_NOW=off

否则当 LD_BIND_NOW 不存在或者为空时，动态链接器允许延迟对过程链接表的求值，这样避免过度
地对那些未调用的函数符号的进行引用解决和重定位。

动态链接段
-----------

如果一个目标文件参与到动态链接中，它的程序头部需要包含一个动态链接分段（PT_DYNAMIC），
该分段包含一个 .dynamic 分区。一个特殊的符号 _DYNAMIC 数组，数组元素是如下的动态链接结
构体： ::

    typedef struct {
        Elf32_Sword d_tag;
        union {
            Elf32_Word d_val;
            Elf32_Addr d_ptr;
        } d_un;
    } Elf32_Dyn;

    extern Elf32_Dyn _DYNAMIC[];

    typedef struct {
        Elf64_Sxword d_tag;
        union {
            Elf64_Xword d_val;
            Elf64_Addr d_ptr;
        } d_un;
    } Elf64_Dyn;

    extern Elf64_Dyn _DYNAMIC[];

其中标签 d_tag 控制着对 d_un 字段的解释，其中 d_val 是一个机器字长的整数值，而 d_ptr
表示程序的虚拟地址。文件的虚拟地址可能与执行时的内存虚拟地址不匹配，当解析这个地址时，动
态链接器会计算实际的地址，基于文件地址和内存基址。

下标列出了可执行文件和共享目标文件对标签 tag 的需求，其中 Mandatory 表示动态链接数组必
须包含该类型的一个条目，而 Option 表示包含这种标签的条目不是必须的。 ::

    名称            值      d_un    可执行文件   共享目标文件
    DT_NULL         0       忽略    mandatory   mandatory
    DT_NEEDED       1       d_val   optional    optional
    DT_PLTRELSZ     2       d_val   optional    optional
    DT_PLTGOT       3       d_ptr   optional    optional
    DT_HASH         4       d_ptr   mandatory   mandatory
    DT_STRTAB       5       d_ptr   mandatory   mandatory
    DT_SYMTAB       6       d_ptr   mandatory   mandatory
    DT_RELA         7       d_ptr   mandatory   optional
    DT_RELASZ       8       d_val   mandatory   optional
    DT_RELAENT      9       d_val   mandatory   optional
    DT_STRSZ        10      d_val   mandatory   mandatory
    DT_SYMENT       11      d_val   mandatory   mandatory
    DT_INIT         12      d_ptr   optional    optional
    DT_FINI         13      d_ptr   optional    optional
    DT_SONAME       14      d_val   忽略        optional
    DT_RPATH*       15      d_val   optional    忽略
    DT_SYMBOLIC*    16      忽略    忽略        optional
    DT_REL          17      d_ptr   mandatory   optional
    DT_RELSZ        18      d_val   mandatory   optional
    DT_RELENT       19      d_val   mandatory   optional
    DT_PLTREL       20      d_val   optional    optional
    DT_DEBUG        21      d_ptr   optional    忽略
    DT_TEXTREL*     22      忽略    optional    optional
    DT_JMPREL       23      d_ptr   optional    optional
    DT_BIND_NOW*    24      忽略    optional    optional
    DT_INIT_ARRAY   25      d_ptr   optional    optional
    DT_FINI_ARRAY   26      d_ptr   optional    optional
    DT_INIT_ARRAYSZ 27      d_val   optional    optional
    DT_FINI_ARRAYSZ 28      d_val   optional    optional
    DT_RUNPATH      29      d_val   optional    optional
    DT_FLAGS        30      d_val   optional    optional
    DT_ENCODING     32 unspecified unspecified unspecified
    DT_PREINIT_ARRAY 32     d_ptr   optional    忽略
    DT_PREINIT_ARRAYSZ 33   d_val   optional    忽略
    DT_SYMTAB_SHNDX 34      d_ptr   optional    optional
    DT_LOOS         0x6000000D unspecified unspecified unspecified
    DT_HIOS         0x6ffff000 unspecified unspecified unspecified
    DT_LOPROC       0x70000000 unspecified unspecified unspecified
    DT_HIPROC       0x7fffffff unspecified unspecified unspecified

DT_NULL
    空元素表示数组的结束
DT_NEEDED
    字符串表的偏移表示所需库的名称，可用有多个依赖库，这些库出现的相对顺序是重要的
DT_PLTRELSZ
    与过程链接表关联的重定位条目的总大小，如果 JMPREL 存在该条目必须存在
DT_PLTGOT
    保存过程链接表或全局偏移表的地址
DT_HASH
    哈希表分区的地址，该哈希表被 SYMTAB 符号表使用
DT_STRTAB
    字符串表分区的地址
DT_SYMTAB
    符号表分区的地址
DT_RELA
    重定位表的地址，每个重定位结构体有一个显式的附加值字段（r_addend）；一个目标文件可
    能有多个重定位分区，但当构建可执行或共享目标文件的重定位表时，链接编辑器会将所有重定
    位分区合并到一个表中，尽管重定位分区在目标文件中是独立的，但是动态链接器看到是一个重
    定位表
DT_RELASZ
    重定位表的总大小
DT_RELAENT
    重定位结构体的大小
DT_STRSZ
    字符串表的总大小
DT_SYMENT
    符号表中符号结构体的大小
DT_INIT
    初始函数的地址
DT_FINI
    终止函数的地址
DT_SONAME
    共享目标文件的名称
DT_RPATH*
    库搜索路径字符串，如果存在 RUNPATH 会被 RUNPATH 覆盖
DT_SYMBOLIC*
    动态链接器首先从共享文件自己内部找未定义符号，失败后才正常查可执行和其他共享文件，会
    被 SYMBOLIC 标记覆盖
DT_REL
    重定位表的地址，与 DT_RELA 类似，但附加值是隐含的
DT_RELSZ
    重定位表的总大小
DT_RELENT
    重定位结构体的大小
DT_PLTREL
    过程链接表引用的重定位表的类型，DT_REL 或 DT_RELA，过程链接表的所有重定位必须是相
    同的类型
DT_DEBUG
    用于调试，其内容未指定
DT_TEXTREL*
    如果存在，重定位条目可以请求修改不可写分段，会被 TEXTREL 标记覆盖
DT_JMPREL
    过程链接表关联的重定位条目的地址，如果存在 PLTSZ 和 PLTREL 必须存在
DT_BIND_NOW*
    告诉动态链接器在把控制权交给程序之前处理所有的重定位，如果不存在则延时绑定或调用
    dlopen(BA_LIB) 绑定，会被 BIND_NOW 标记覆盖
DT_INIT_ARRAY
    初始函数指针数组的地址
DT_FINI_ARRAY
    终结函数指针数组的地址
DT_INIT_ARRAYSZ
    初始函数指针数组的大小
DT_FINI_ARRAYSZ
    终结函数指针数组的大小
DT_RUNPATH
    库搜索路径字符串，会覆盖 RPATH
DT_FLAGS
    动态加载标记
DT_ENCODING
    未指定
DT_PREINIT_ARRAY
    预初始化函数指针数组的地址
DT_PREINIT_ARRAYSZ
    预初始化函数指针数组的大小
DT_SYMTAB_SHNDX
    符号表 DT_SYMTAB 引用的 SHT_SYMTAB_SHNDX 分区的地址
DT_LOOS
    未指定
DT_HIOS
    未指定
DT_LOPROC
    未指定
DT_HIPROC
    未指定

还有一个名称 DT_JUMP_REL 是历史遗留名称。

动态链接标记 DT_FLAGS 可以指定以下这些值： ::

    DF_ORIGIN       0x1
    DF_SYMBOLIC     0x2
    DF_TEXTREL      0x4
    DF_BIND_NOW     0x8
    DF_STATIC_TLS   0x10

DF_ORIGIN
    加载的目标可以引用 $ORIGIN 替换字符串，动态链接器必须在目标加载时确定包含这个条目的
    目标的路径名称
DF_SYMBOLIC
    用于共享目标文件，动态链接器会首先使用共享目标文件内部的符号来解析未定义符号
DF_TEXTREL
    重定位条目可以请求修改不可写分段
DF_BIND_NOW
    动态链接器在把控制权交给程序之前处理所有的重定位，否则延时绑定或调用 dlopen(BA_LIB) 
    绑定
DF_STATIC_TLS
    让动态链接器拒绝动态加载 TLS，表示使用的是静态 TLS

共享目标依赖
-------------

当链接编辑器处理归档目标库时，会解压库中的成员文件并拷贝到输出目标文件中。这些静态服务在
执行时不需要涉及动态链接器。共享目标也可以提供服务，但是需要动态链接器将对应的共享目标文
件附加到程序的进行映像中。

当动态链接器创建一个目标文件的内存分段时，该目标文件依赖的库（由 DT_NEEDED 类型的动态链
接结构体提供）告诉链接器需要额外加载哪些依赖库。动态链接器通过不断加载共享对象以及依赖的
库、依赖库依赖的库，最终创建一个完整的进程映像。在解决符号引用时，动态链接器使用广度优先
算法对符号表进行搜索，即先查找可执行程序自己的符号表，然后依次查找依赖的共享库中的符号表，
再查找共享库依赖的共享库中的符号表。即使一个共享库可能被引用多次，但是动态链接器只能加载
一次。

动态链接结构体中保存的动态库名称，要么时从共享目标文件的 DT_SONAME 拷贝而来，要么时共享
目标文件的路径名称。例如，如果链接编辑器使用一个 DT_SONAME 值为 lib1 的共享目标文件和
另外一个路径为 /usr/lib/lib2 的共享目标文件创建可执行文件，可执行文件会包含 lib1 和
/usr/lib/lib2 两个依赖共享目标文件。

如果一个共享文件名称包含一个或多个斜杠字符，例如 /usr/lib/lib2 或者 dir/file，动态链
接器直接使用这个名称作为路径名。如果不包含斜杠字符，例如 lib1，动态链接器会在以下三个目
录搜索该共享库：

1. DT_RUNPATH 指定的以冒号分隔的目录，例如 /home/dir/lib:/home/dir2/lib: 会搜索
   /home/dir/lib 目录，再搜索 /home/dir2/lib 目录，再搜索当前目录；DT_RUNPATH 只会
   影响可执行文件或共享文件的直接依赖的查找，即只对当前文件中的 DT_NEEDED 依赖库有影响
2. 进程环境中的变量 LD_LIBRARY_PATH 以冒号分隔的目录，处于安全动态链接器会忽略设置了用
   户和组ID的 LD_LIBRARY_PATH，但时 DT_RUNPATH 和默认目录仍然会搜索
3. 搜索默认目录，例如 /usr/lib 或处理器补充文档指定的目录

当动态链接器查找共享目标文件时，需要确定根据 ELF 头部信息来确定是否时一个合法的 ELF 文
件，如果不是继续查找下一个。需要检查的头部信息包括 e_ident[EI_DATA]，e_ident[EI_CLASS]，
e_ident[EI_OSABI]，e_ident[EI_ABIVERSION]，e_machine，e_type，e_flags 以及 e_version。

替换序列
---------

DT_NEEDED 或 DT_RUNPATH 动态链接结构体提供的路径字符串，以及传给过程 dlopen() 的路径
名称，可以包含 $ 符号来引入一个替换序列。这个序列以 $ 符号开始，后面取最长标识符名称或
者名称包含在大括号内，名称只能以字母和下划线开始跟随零个或多个字母、数字、下划线。如果 $
符号没有紧跟一个标识符名称或大括号，它的行为没有明确指定。

如果名称是 ORIGIN，替换序列会被动态链接器替换成当前目标文件所在目录的绝对路径，并且路径
中不会包含符号链接或者使用 . 或者 .. 目录名。如果名称不是 ORIGIN，那么动态链接器的行为
没有指定。

当动态链接器加载一个包含 $ORIGIN 的目标文件时，它必须确定当前目标文件所在目录的路径。因
为确定路径是昂贵的，具体实现会尽量避免去确认路径。例如，当目标使用带 $ORIGIN 的路径调用
dlopen()，但自己的动态链接结构体又没有使用 $ORIGIN 时，动态链接器可能直到 dlopen() 真
正调用时才计算 $ORIGIN 的路径。但是应用程序可能在调用 dlopen() 之前切换了当前的工作路
径，这种计算可能时不对的。为了避免这种情况，目标文件需要在动态链接结构体中使用 DF_ORIGIN
标记。当目标文件没有设置 DF_ORIGIN 标记并且没有在动态链接结构体中使用 $ORIGIN 时，具体
实现可能拒绝 dlopen() 中提供的 $ORIGIN 参数。

为了安全，动态链接器不允许设置了用户和组ID的程序使用 $ORIGIN 替换。这种情况下，动态链接
器会忽略这种搜索路径，或 DT_DEEDED 和 dlopen() 中使用 $ORIGIN 会被当作错误。

全局偏移表
-----------

见处理器补充规范。

过程链接表
-----------

见处理器补充规范。

哈希表
-------

哈希表的内容是一个 ElfWord 整数数组，它的组织方式示意如下： ::

    nbucket
    nchain
    bucket[0]
    ...
    bucket[nbucket-1]
    chain[0]
    ...
    chain[nchain-1]

使用的哈希函数如下： ::

    // 传入符号名称，返回用于计算桶索引的值
    uint32 elf_hash(const byte* sym_name) {
        uint32 h = 0, g;
        while (*sym_name) {
            h = (h << 4) + *sym_name++;
            g = (h & 0xf0000000);
            if (g) {
                h ^= g >> 24;
            }
            h &= ~g;
        }
        return h;
    }

nbucket
    桶的个数，bucket[elf_hash(sym_name)%nbucket] 存储的是符号索引
nchain
    链接的个数，必须等于符号表中符号的个数
bucket
    通过 elf_hash 哈希函数计算符号对应的桶索引，对应桶索引中保存的是该符号在符号表中的
    索引，符号表定义在 DT_SYMTAB 动态链接结构体中
chain
    该数组保存的也是符号表索引，数组大小是nchain，即符号表中符号的个数，因此符号表索引
    也索引 chain 表的元素。bucket[elf_hash(sym_name)%nbucket] 计算出的索引 y 同时索
    引符号表和 chain 表，如果 y 索引的符号匹配失败，继续匹配 chain[y]、chain[y+1] 对
    应的符号名称，直到匹配成功或 chain 元素的值为 STN_UNDEF。

初始和终止函数
--------------

当动态链接器加载进程映像并执行重定位后，就可以开始执行共享对象以及可执行文件的初始化函
数。所有共享对象的初始化会在可执行文件获得控制权之前执行，而可执行文件的终止函数会在所
有共享对象的终止函数执行前执行。

在目标 A 的初始函数执行之前，目标 A 依赖的所有其他目标的初始函数会先执行。如果目标 A 依
赖另一目标 B，那么 B 需要出现在 A 的依赖的目标列表中（DT_NEEDED 类型的动态链接结构体）。
但是目标列表中哪个目标先执行没有指定。同样的，共享目标和可执行文件可能有终止函数，终止函
数由过程 atexit(BA_OS) 提供，目标 A 的终止函数必须在它依赖的所有其他目标的终止函数执行
前执行。

一个可执行文件还可能有预初始化函数，这些函数在所有初始化函数执行前执行。共享目标不允许有
预初始化函数。但是在执行预初始化时，系统库的初始化可能还没有执行，因此预初始化的代码不能
依赖于系统库。

另外，动态链接器保证不会重复执行任何预初始化函数、初始化函数、以及终止函数。当执行一个对
象的初始函数时，DT_PREINIT_ARRAY 会先按顺序执行，然后是 DT_INIT 函数，最后是 DT_INIT_ARRAY
按顺序执行；当执行对象的终止函数时，DT_FINI_ARRAY 中的函数会先倒序执行，然后执行 DT_FINI
函数。

但是过程 atexit(BA_OS) 提供的终止函数不保证总是被执行，例如当进程执行死机时，或者调用
exit(BA_OS) 终止进程，或者接收到信号或异常但没有处理时。
