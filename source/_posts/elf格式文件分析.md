---
title: elf格式文件分析
date: 2023-01-07 21:21:07
tags: elf
categories: elf
top_img: https://s2.loli.net/2023/01/07/W8nPCv2isQMcrTp.png
cover: https://s2.loli.net/2023/01/07/W8nPCv2isQMcrTp.png
---

![](https://s2.loli.net/2023/01/07/W8nPCv2isQMcrTp.png)

![](https://s2.loli.net/2023/01/10/tEPHR4Nd1bXG2j9.png)



# 1 简介

  可执行与可链接格式 （Executable and Linkable Format，ELF），常被称为 ELF格式，是一种用于可执行文件、目标代码、共享库和核心转储（core dump）的标准文件格式，一般用于类Unix系统，比如Linux，Macox等。ELF 格式灵活性高、可扩展，并且跨平台。比如它支持不同的字节序和地址范围，所以它不会不兼容某一特别的 CPU 或指令架构。这也使得 ELF 格式能够被运行于众多不同平台的各种操作系统所广泛采纳。
  ELF文件一般由三种类型的文件：

- 可重定向文件：文件保存着代码和适当的数据，用来和其他的目标文件一起来创建一个可执行文件或者是一个共享目标文件。比如编译的中间产物`.o`文件；
- 可执行文件：一个可执行文件；
- 共享目标文件：共享库。文件保存着代码和合适的数据，用来被下连接编辑器和动态链接器链接。比如linux下的`.so`文件。

# 2 ELF文件格式

## 2.1 ELF Header

ELF文件头描述了ELF文件的基本类型，地址偏移等信息，分为32bit和64bit两个版本，定义于linux源码的`/usr/include/elf.h`文件中。

```c++
#define EI_NIDENT	16

typedef struct elf32_hdr{
  unsigned char	e_ident[EI_NIDENT];//elf 标识
  Elf32_Half	e_type;
  Elf32_Half	e_machine;
  Elf32_Word	e_version;
  Elf32_Addr	e_entry;  /* Entry point 程序入口地址*/ 
  Elf32_Off	e_phoff; //Program header table 文件中的偏移
  Elf32_Off	e_shoff; //Section header table 文件中的偏移
  Elf32_Word	e_flags;
  Elf32_Half	e_ehsize;//elfheader头占的大小
  Elf32_Half	e_phentsize; //pht 每个表的占的大小
  Elf32_Half	e_phnum;//pht 表的个数
  Elf32_Half	e_shentsize; //sht 每个表的占的大小
  Elf32_Half	e_shnum;//sht 表的个数
  Elf32_Half	e_shstrndx;//字符串表在sht表的下标
} Elf32_Ehdr;

typedef struct elf64_hdr {
  unsigned char	e_ident[EI_NIDENT];	/* ELF "magic number" */
  Elf64_Half e_type;
  Elf64_Half e_machine;
  Elf64_Word e_version;
  Elf64_Addr e_entry;		/* Entry point virtual address */
  Elf64_Off e_phoff;		/* Program header table file offset */
  Elf64_Off e_shoff;		/* Section header table file offset */
  Elf64_Word e_flags;
  Elf64_Half e_ehsize;
  Elf64_Half e_phentsize;
  Elf64_Half e_phnum;
  Elf64_Half e_shentsize;
  Elf64_Half e_shnum;
  Elf64_Half e_shstrndx;
} Elf64_Ehdr;


```

从上面的结构中能够看出32bit和64bit的区别仅仅是字长的区别，字段上没有实际上的差别。每个字段的含义如下：

- e_ident：ELF文件的描述，是一个16字节的标识，表明当前文件的数据格式，位数等：
  - [0,3]字节为魔数，即`e_ident[EI_MAG0-EI_MAG3]`，取值为固定的`0x7f E L F`，标记当前文件为一个ELF文件；
  - [4,4]字节为EI_CLASS即e_ident[EI_CLASS]，表明当前文件的类别：
    - 0：表示非法的类别；
    - 1：表示32bit；
    - 2：表示64bit；
  - [5,5]字节为EI_DATA即e_ident[EI_DATA],表明当期那文件的数据排列方式：
    - 0表示非法；
    - 1表示小端；
    - 2表示大端；
  - [6,6]字节为`EI_VERSION`即`e_ident[EI_VERSION]`，表明当前文件的版本，目前该取值必须为`EV_CURRENT`即1；
  - [7,7]字节为`EI_PAD`即`e_ident[EI_PAD]`表明`e_ident`中未使用的字节的起点（值是相对于`e_ident[EI_PAD+1]`的偏移），未使用的字节会被初始化为0，解析ELF文件时需要忽略对应的字段；

> ```
>   EI_MAG0,EI_MAG1,EI_MAG2,EI_MAG3,EI_CLASS,EI_DATA,EI_VERSION，EI_OSABI,EI_PAD是linux源码中定义的宏，取值分别为0-7，分别对应各个字段的下标；下面的[宏定义](https://so.csdn.net/so/search?q=宏定义&spm=1001.2101.3001.7020)将采用类似EI_MAG0(0)的方式，表示EI_MAG0的值为0。
> ```
>
> 

- e_type：文件的标识字段标识文件的类型；
  - `ET_NONE(0)`：未知的文件格式；
  - `ET_REL(1)`：可重定位文件，比如目标文件；
  - `ET_EXEC(2)`：可执行文件；
  - `ET_DYN(3)`：共享目标文件；
  - `ET_CORE(4)`：Core转储文件，比如程序crash之后的转储文件；
  - `ET_LOPROC(0xff00)`：特定处理器的文件标识；
  - `ET_HIPROC(0xffff)`：特定处理器的文件标识;
  - `[ET_LOPROC,ET_HIPROC]`之间的值用来表示特定处理器的文件格式；
- e_machine：目标文件的体系结构（下面列举了少数处理器架构，具体ELF文件支持的架构在对应的文件中查看即可）；
  - `ET_NONE(0)`：未知的处理器架构；
  - `EM_M32(1)`：AT&T WE 32100；
  - `EM_SPARC(2)`：SPARC；
  - `EM_386(3)`：Intel 80386；
  - `EM_68K(4)`：Motorola 68000；
  - `EM_88K(5)`：Motorola 88000；
  - `EM_860(6)`：Intel 80860；
  - `EM_MIPS(7)`：MIPS RS3000大端；
  - `EM_MIPS_RS4_BE(10)`：MIPS RS4000大端；
  - 其他，预留；
- e_version：当前文件的版本；
  - `EV_NONE(0)`：非法的版本；
  - `EV_CURRENT(`)`：当前版本；
- `e_entry`：程序的虚拟入口地址，如果文件没有对应的入口可以为0；
- `e_phoff`：文件中程序头表的偏移（bytes），如果文件没有该项，则应该为0；
- `e_shoff`：文件中段表/节表的偏移（bytes），如果文件没有该项，则应该为0；
- `e_flags`：处理器相关的标志位，宏格式为`EF_machine_flag`比如`EF_MIPS_PIC`；
- `e_ehsize`：ELF文件头的大小（bytes）；
- `e_phentsize`：程序头表中单项的大小，表中每一项的大小相同；
- `e_phnum`：程序头表中的项数，也就是说程序头表的实际大小为`ephentsize x e_phnum`，如果文件中没有程序头表该项为0；
- `e_shentsize`：节表中单项的大小，表中每一项的大小相同；
- `e_shnum`：节表中项的数量；
- `e_shstrndx`：节表中节名的索引，如果文件没有该表则该项为`SHN_UNDEF(0)`。



## 2.2 程序头表（Program Header Table）

可执行文件或者共享目标文件的程序头部是一个结构数组，每个结构描述了一个段 或者系统准备程序执行所必需的其它信息。程序头表描述了ELF文件中Segment在文件中的布局，描述了OS该如何装载可执行文件到内存。程序头表的表项的描述如下，类似于ELF Header也有32和64位两个版本。

```c
typedef struct elf32_phdr {
	Elf32_Word p_type;   //当前Segment的类型；
	Elf32_Off p_offset;
	Elf32_Addr p_vaddr;
	Elf32_Addr p_paddr;
	Elf32_Word p_filesz;
	Elf32_Word p_memsz;
	Elf32_Word p_flags;
	Elf32_Word p_align;
} Elf32_Phdr;
typedef struct elf64_phdr {
	Elf64_Word p_type;
	Elf64_Word p_flags;
	Elf64_Off p_offset;	/* Segment file offset */
	Elf64_Addr p_vaddr;	/* Segment virtual address */
	Elf64_Addr p_paddr;	/* Segment physical address */
	Elf64_Xword p_filesz;	/* Segment size in file */
	Elf64_Xword p_memsz;	/* Segment size in memory */
	Elf64_Xword p_align;	/* Segment alignment, file & memory */
} Elf64_Phdr

```

- p_type ：当前Segment的类型；

  - `PT_NULL(0)`：当前项未使用，项中的成员是未定义的，需要忽略当前项；
  - `PT_LOAD(1)`：当前Segment是一个可装载的Segment，即可以被装载映射到内存中，其大小由`p_filesz`和`p_memsz`描述。如果`p_memsz>p_filesz`则剩余的字节被置零，但是`p_filesz>p_memsz`是非法的。动态库一般包含两个该类型的段：代码段和数据段；
  - **`PT_DYNAMIC(2)`：动态段，动态库特有的段，包含了动态链接必须的一些信息，比如需要链接的共享库列表、GOT等等；**
  - `PT_INTERP(3)`：当前段用于存储一段以NULL为结尾的字符串，该字符串表明了程序解释器的位置。且当前段仅仅对于可执行文件有实际意义，一个可执行文件中不能出现两个当前段，如果一个文件中包含当前段。比如`/lib64/ld-linux-x86-64.so.2`；
  - `PT_NOTE(4)`：用于保存与特定供应商或者系统相关的附加信息以便于兼容性、一致性检查，但是实际上只保存了操作系统的规范信息；
  - `PT_SHLIB(5)`：保留段；
  - `PT_PHDR(6)`：保存程序头表本身的位置和大小，当前段不能在文件中出现一次以上，且仅仅当程序表头为内存映像的一部分时起作用，它必须在所有加载项目之前；
  - `[PT_LPROC(0x70000000),PT_HIPROC(0x7fffffff)]`：该范围内的值用作预留；

- `p_offset`：当前段相对于文件起始位置的偏移量；

- `p_vaddr`：段的第一个字节将被映射到到内存中的虚拟地址；

- `p_paddr`：此成员仅用于与物理地址相关的系统中。因为 System V 忽略所有应用程序的物理地址信息，此字段对与可执行文件和共享目标文件而言具体内容是指定的；

- `p_filesz`：段在文件映像中所占的字节数，可能为 0；

- `p_memsz`：段在内存映像中占用的字节数，可能为 0；

- `p_flags`：段相关的标志；

- p_align：段在文件中和内存中如何对齐。可加载的进程段的

  p_vaddr和-p_offset 取值必须合适，相对于对页面大小的取模而言；

  - 0和1表示不需要对齐；
  - 其他值必须为2的幂次方，且必须 p\_addr|p\_align==p\_offset| palign

## 2.3 节头表（Section Header Table）

节头表描述了ELF文件中的节的基本信息。可执行文件不一定由节头表但是一定有节，节头表可利用特殊的方式去除。

**段和节的区别是：**

- **段包含了程序装载可执行的基本信息，段告诉OS如何装载当前段到虚拟内存以及当前段的权限等和执行相关的信息，一个段可以包含0个或多个节；**
- **节包含了程序的代码和数据等内容，链接器会将多个节合并为一个段。**

```c
typedef struct elf32_shdr {
  Elf32_Word	sh_name;
  Elf32_Word	sh_type;
  Elf32_Word	sh_flags;
  Elf32_Addr	sh_addr;
  Elf32_Off	sh_offset;
  Elf32_Word	sh_size;
  Elf32_Word	sh_link;
  Elf32_Word	sh_info;
  Elf32_Word	sh_addralign;
  Elf32_Word	sh_entsize;
} Elf32_Shdr;
typedef struct elf64_shdr {
  Elf64_Word sh_name;		/* Section name, index in string tbl */
  Elf64_Word sh_type;		/* Type of section */
  Elf64_Xword sh_flags;		/* Miscellaneous section attributes */
  Elf64_Addr sh_addr;		/* Section virtual addr at execution */
  Elf64_Off sh_offset;		/* Section file offset */
  Elf64_Xword sh_size;		/* Size of section in bytes */
  Elf64_Word sh_link;		/* Index of another section */
  Elf64_Word sh_info;		/* Additional section information */
  Elf64_Xword sh_addralign;	/* Section alignment */
  Elf64_Xword sh_entsize;	/* Entry size if section holds table */
} Elf64_Shdr;

```

- `sh_name`：值是节名称在字符串表中的索引；
- sh_type：描述节的类型和语义；
  - `SHT_NULL(0)`：当前节是非活跃的，没有一个对应的具体的节内存；
  - `SHT_PROGBITS(1)`：包含了程序的指令信息、数据等程序运行相关的信息；
  - SHT_SYMTAB(2)：保存了符号信息，用于重定位；
    - 此种类型节的`sh_link`存储相关字符串表的节索引，`sh_info`存储最后一个局部符号的符号表索引+1；
  - SHT_DYNSYM(11)：保存共享库导入动态符号信息；
    - 此种类型节的`sh_link`存储相关字符串表的节索引，`sh_info`存储最后一个局部符号的符号表索引+1；
  - `SHT_STRTAB(3)`：一个字符串表，保存了每个节的节名称；
  - SHT_RELA(4)：存储可重定位表项，可能会有附加内容，目标文件可能有多个可重定位表项；
    - 此种类型节的`sh_link`存储相关符号表的节索引，`sh_info`存储重定位所使用节的索引；
  - SHT_HASH(5)：存储符号哈希表，所有参与动态链接的目标只能包含一个哈希表，一个目标文件只能包含一个哈希表；
    - 此种类型节的`sh_link`存储哈希表所使用的符号表的节索引,`sh_info`为0；
  - SHT_DYAMIC(6)：存储包含动态链接的信息，一个目标文件只能包含一个；
    - 此种类型的节的`sh_link`存储当前节中使用到的字符串表格的节的索引，`sh_info`为0；
  - `SHT_NOTE(7)`：存储以某种形式标记文件的信息；
  - `SHT_NOBITS(8)`：这种类型的节不占据文件空间，但是成员`sh_offset`依然会包含对应的偏移；
  - SHT_REL(9)：包含可重定位表项，无附加内容，目标文件可能有多个可重定位表项；
    - 此种类型节的`sh_link`存储相关符号表的节索引，`sh_info`存储重定位所使用节的索引；
  - `SHT_SHLIB(10)`：保留区，包含此节的程序与ABI不兼容；
  - `[SHT_LOPROC(0x70000000),SHT_HIPROC(0x7fffffff)]`：留给处理器专用语义；
  - `[SHT_LOUSER(0x80000000),SHT_HIUSER(0xffffffff)]`：预留；
- sh_flags：1bit位的标志位；
  - `SHF_WRITE(0x1)`：当前节包含进程执行过程中可写的数据；
  - `SHF_ALLOC(0x2)`：当前节在运行阶段占据内存；
  - `SHF_EXECINSTR(0x4)`：当前节包含可执行的机器指令；
  - `SHF_MASKPROC(0xf0000000)`：所有包含当前掩码都表示预留给特定处理器的；
- `sh_addr`：如果当前节需要被装载到内存，则当前项存储当前节映射到内存的首地址，否则应该为0；
- `sh_offset`：当前节的首地址相对于文件的偏移；
- `sh_size`：节的大小。但是对于类型为`SHT_NOBITS`的节，当前值可能不为0但是在文件中不占据任何空间；
- `sh_link`：存储节投标中的索引，表示当前节依赖于对应的节。对于特定的节有特定的含义，其他为`SHN_UNDEF`；
- `sh_info`：节的附加信息。对于特定的节有特定的含义，其他为`0`；
- `sh_addralign`：地址约束对齐，值应该为0或者2的幂次方，0和1表示未进行对齐；
- `sh_entsize`：某些节是一个数组，对于这类节当前字段给出数组中每个项的字节数，比如符号表。如果节并不包含对应的数组，值应该为0。

## 2.3 一些特殊的节

  ELF文件中有一些预定义的节来保存程序、数据和一些控制信息，这些节被用来链接或者装载程序。每个操作系统都支持一组链接模式，主要分为两类（也就是常说的动态库和静态库）：

- Static：静态绑定的一组目标文件、系统库和库档案（比如静态库），解析包含的符号引用并创建一个完全自包含的可执行文件；
- Dynamic：一组目标文件、库、系统共享资源和其他共享库链接在一起创建可执行文件。当加载此可执行文件时必须使系统中其他共享资源和动态库可用，程序才能正常运行。

  库文件无论是动态库还是静态库在其文件中都包含对应的节，一些特殊的节其功能如下：

- `.bss`，类型`SHT_NOBITS`，属性`SHF_ALLOC|SHF_WRITE`：存储未经初始化的数据。根据定义程序开始执行时，系统会将这些数据初始化为0，且此节不占用文件空间；
- `.comment`，类型`SHT_PROGBITS`，属性`none`：存储版本控制信息；
- `.data`，类型`SHT_PROGBITS`，属性`SHF_ALLOC|SHF_WRITE`：存放初始化的数据；
- `.data1`，类型`SHT_PROGBITS`，属性`SHF_ALLOC|SHF_WRITE`：存放初始化的数据；
- `.debug`，类型`SHT_PROGBITS`，属性`none`：存放用于符号调试的信息；
- `.dynamic`，类型`SHT_DYNAMIC`，属性`SHF_ALLOC`，是否有属性`SHF_WRITE`屈居于处理器：包含动态链接的信息，
- `.hash`，类型`SHT_HASH`，属性`SHF_ALLOC`：
- `.line`，类型`SHT_PROGBITS`，属性`none`：存储调试的行号信息，描述源代码和机器码之间的对应关系；
- `.note`，类型`SHT_NOTE`，属性`none`：
- `.rodata`，类型`SHT_PROGBITS`，属性`SHF_ALLOC`：存储只读数据；
- `.rodata1`，类型`SHT_PROGBITS`，属性`SHF_ALLOC`：存储只读数据；
- `.shstrtab`，类型`SHT_STRTAB`，属性`none`：存储节的名称；
- `.strtab`，类型`SHT_STRTAB`：存储常见的与符号表关联的字符串。如果文件有一个包含符号字符串表的可加载段，则该段的属性将包括 SHF_ALLOC 位； 否则，该位将关闭；
- `.symtab`，类型`SHT_SYMTAB`，属性``````：存储一个符号表。如果文件具有包含符号表的可加载段，则该节的属性将包括 SHF_ALLOC 位；否则，该位将关闭；
- `.text`，类型`SHT_PROGBITS`，属性`SHF_ALLOC|SHF_EXECINSTR`：存储程序的代码指令；
- `.dynstr`，类型`SHT_STRTAB`，属性`SHF_ALLOC`：存储动态链接所需的字符串，最常见的是表示与符号表条目关联的名称的字符串；
- `.dynsym`，类型`SHT_DYNSYM`，属性`SHF_ALLOC`：存储动态链接符号表；
- `.fini`，类型`SHT_PROGBITS`，属性`SHF_ALLOC|SHF_EXECINSTR`：存储有助于进程终止代码的可执行指令。 当程序正常退出时，系统执行本节代码；
- `.init`，类型`SHT_PROGBITS`，属性`SHF_ALLOC|SHF_EXECINSTR`：存储有助于进程初始化代码的可执行指令。 当程序开始运行时，系统会在调用主程序入口点（C 程序称为 main）之前执行本节中的代码；
- `.interp`，类型`SHT_PROGBITS`：保存程序解释器的路径名。 如果文件有一个包含该节的可加载段，则该节的属性将包括 SHF_ALLOC 位； 否则，该位将关闭；
- `.relname`，类型`SHT_REL`：包含重定位信息。如果文件具有包含重定位的可加载段，则这些部分的属性将包括 SHF_ALLOC 位；否则，该位将关闭。通常，名称由 重定位适用的部分。因此`.text`的重定位部分通常具有名称.`rel.text`或`.rela.text`；
- `.relaname`，类型`SHT_RELA`：同`relname`。
- 其他：对于C++程序有些版本会有`.ctors`（有时也会是`.init_array`，见[Can’t find .dtors and .ctors in binary](https://stackoverflow.com/questions/16569495/cant-find-dtors-and-ctors-in-binary)）和`dtors`两个节存储构造和析构相关的代码。

>   带有点 (.) 前缀的部分名称是为系统保留的，但如果它们的现有含义令人满意，应用程序可以使用这些部分。 应用程序可以使用不带前缀的名称以避免与系统部分冲突。 目标文件格式允许定义不在上面列表中的部分。 一个目标文件可能有多个同名的部分。

## 2.4 字符串表

  字符串表是一个存储字符串的表格，而每个字符串是以NULL也就是`\0`为结尾的。字符串表格中索引为0处的字符串被定义为空字符串。符号表中保存的字符串是节名和目标文件中使用到的符号。而需要使用对应字符串时，只需要在需要使用的地方指明对应字符在字符串表中的索引即可，使用的字符串就是索引处到第一个`\0`之间的字符串。
![在这里插入图片描述](https://img-blog.csdnimg.cn/c069cbc6a3f9496ca578e856d7f5a421.png)

## 2.5 符号表

  目标文件的符号表包含定位和重定位程序的符号定义和引用所需的信息。符号表索引是该数组的下标。索引0既指定表中的第一个条目，又用作未定义的符号索引。

```c
typedef struct elf32_sym{
  Elf32_Word	st_name;
  Elf32_Addr	st_value;
  Elf32_Word	st_size;
  unsigned char	st_info;
  unsigned char	st_other;
  Elf32_Half	st_shndx;
} Elf32_Sym;
12345678
typedef struct elf64_sym {
  Elf64_Word st_name;		/* Symbol name, index in string tbl */
  unsigned char	st_info;	/* Type and binding attributes */
  unsigned char	st_other;	/* No defined meaning, 0 */
  Elf64_Half st_shndx;		/* Associated section index */
  Elf64_Addr st_value;		/* Value of the symbol */
  Elf64_Xword st_size;		/* Associated symbol size */
} Elf64_Sym;
12345678
```

- `st_name`：存储一个指向字符串表的索引来表示对应符号的名称；
- st_value：存储对应符号的取值，具体值依赖于上下文，可能是一个指针地址，立即数等。另外，不同对象文件类型的符号表条目对 st_value 成员的解释略有不同：
  - 在重定位文件中在可重定位文件中，`st_value`保存节索引为`SHN_COMMON`的符号的对齐约束；
  - 在可重定位文件中，`st_value`保存已定义符号的节偏移量。 也就是说，`st_value`是从`st_shndx`标识的部分的开头的偏移量；
  - 在可执行文件和共享对象文件中，`st_value`保存一个虚拟地址。 为了使这些文件的符号对动态链接器更有用，节偏移（文件解释）让位于与节号无关的虚拟地址（内存解释）。
- `st_size`：符号的大小，具体指为`sizeof(instance)`，如果未知则为0；
- `st_info`：指定符号的类型和绑定属性。可以用下面的代码分别解析出`bind,type,info`三个属性：

```c
#define ELF32_ST_BIND(i) ((i)>>4) 
#define ELF32_ST_TYPE(i) ((i)&0xf) 
#define ELF32_ST_INFO(b,t) (((b)<<4)+((t)&0xf))
123
```

- BIND
  - `STB_LOCAL(0)`：局部符号在包含其定义的目标文件之外是不可见的。 同名的本地符号可以存在于多个文件中，互不干扰；
  - `STB_GLOBAL(1)`：全局符号对所有正在组合的目标文件都是可见的。 一个文件对全局符号的定义将满足另一个文件对同一全局符号的未定义引用；
  - `STB_WEAK(2)`：弱符号类似于全局符号，但它们的定义具有较低的优先级；
  - `[STB_LOPROC(13),STB_HIPROC(15)]`：预留位，用于特殊处理器的特定含义；
- TYPE：
  - `STT_NOTYPE(0)`：符号的类型未指定；
  - `STT_OBJECT(1)`：符号与数据对象相关联，例如变量、数组等；
  - `STT_FUNC(2)`：符号与函数或其他可执行代码相关联；
  - `STT_SECTION(3)`：该符号与一个节相关联。 这种类型的符号表条目主要用于重定位，通常具有`STB_LOCAL`BIND属性；
  - `STT_FILE(4)`：一个有`STB_LOCAL`的BIND属性的文件符号的节索引为`SHN_ABS`。并且如果存在其他`STB_LOCAL`属性的符号，则当前符号应该在其之前；
  - `[STT_LOPROC(13),STT_HIPROC(15)]`：预留位，用于特殊处理器的特定含义；
- INFO：
  - `SHN_ABS`：符号有一个绝对值，不会因为重定位而改变；
  - `SHN_COMMON`：该符号标记尚未分配的公共块。 符号的值给出了对齐约束，类似于节的 sh_addralign 成员。 也就是说，链接编辑器将为符号分配存储空间，该地址是 st_value 的倍数。 符号的大小表明需要多少字节；
  - `SHN_UNDEF`：此节表索引表示该符号未定义。 当链接编辑器将此对象文件与另一个定义指定符号的文件组合时，此文件对符号的引用将链接到实际定义；
- `st_other`：该成员当前持有 0 并且没有定义的含义；
- `st_shndx`：每个符号都有属于的节，当前成员存储的就是对应节的索引。

Segment --->Program header table

- 用于告诉内核,在执行ELF文件时应该如何映射内存
- 每个Segment主要包含加载地址,文件中的范围,内存权限,对齐等信息
- 是运行时必须提供的信息

Section ---> Section header table

- 用于告诉链接器,ELF中每个部份是什么,哪里是代码,哪里是只读数据,哪里是重定位信息
- 每个Section主要包含Section类型.文件中的位置,大小等信息
- 链接器依赖Section信息将不同的对象文件的代码,数据信息合并,并修复互相引用

Segment与Section的关系

- 相同对限的Section会放入同一个Segment,例如.text和.rodata section
- 一个Segment包含许多Section,一个Section可以属于多个Segment

# 3 ELF文件格式

## 3.1感染ELF文件注入实现

```
1. 在dt_strtab指向的字符串中添加自定义so模块名称，将字符串表移动到文件末尾
2. 添加一个pt_load表，用于能够内存映射我们添加的字符串，phdr移动到文件末尾
3. 修改dt_strtab,dt_strsz,添加dt_needed
4. 修改ELF Header关于段头表的信息字段，偏移，大小
```

```c++
/*                                                                                                  
 * Description: 修改内存中Elf文件的数据buff，使其在运行时加载自定义模块（修改ELF文件的段头表信息）                                      
 * Input:       pbyElfFile为Elf文件数据buff, ulFileSize为文件大小                                             
 * Output:      ppbyDataBuff为加载到内存后的buff指针                                                          
 * Return:      修改之后的ELF文件大小                                                                        
 * Others:      无                                                                                   
 */                                                                                                 
uint32_t ModifyElfFile(char *pbyElfFile, uint32_t ulFileSize) {                                     
    //将pEhdr指针定位到Phdr数据                                                                             
    Elf32_Ehdr *pEhdr = (Elf32_Ehdr *) pbyElfFile;                                                  
    if (!IS_ELF(*pEhdr)) {                                                                          
        printf("[ERROR] elf magic error!\n");                                                       
        exit(-3);                                                                                   
    }                                                                                               
                                                                                                    
    //将程序头表移动到文件尾以便新增表项                                                                             
    Elf32_Phdr *pPhdr = (Elf32_Phdr *) (pbyElfFile +                                                
                                        pEhdr->e_phoff);   //得到Program header table 在elf文件中的偏移量     
    uint32_t ulPhdrSize = pEhdr->e_phnum * pEhdr->e_phentsize;    //e_phnum 多少个表  e_phentsize每个表的长度 
    memcpy(pbyElfFile + ulFileSize, pPhdr, ulPhdrSize); //把整个Program header talbe 放到尾部              
    pPhdr = (Elf32_Phdr *) (pbyElfFile + ulFileSize);                                               
    printf("[INFO] move phdr success!\n");                                                          
                                                                                                    
    //增加一个程序头表项                                                                                     
    Elf32_Phdr *pPTLoad = pPhdr + pEhdr->e_phnum;                                                   
    pEhdr->e_phoff = ulFileSize;    //修昨段头表的偏移地址                                                    
    pEhdr->e_phnum += 1;         //段头表加1                                                            
    ulPhdrSize += pEhdr->e_phentsize;    //加一个表的长度                                                  
    printf("[INFO] add phdr ent success!\n");                                                       
                                                                                                    
    //将DT_STRTAB移到文件尾并且添加新so的名字                                                                   
    char *pOldStrTab = NULL;                                                                        
    uint32_t ulOldStrTabLen = 0;                                                                    
    int i = 0;                                                                                      
    for (i = 0; i < pEhdr->e_phnum; i++) {                                                          
        if (pPhdr[i].p_type == PT_DYNAMIC) {                                                        
            Elf32_Dyn *pDyn = (Elf32_Dyn *) (pbyElfFile + pPhdr[i].p_offset);  //获取到dynamic segment 
            for (; (char *) pDyn - (pbyElfFile + pPhdr[i].p_offset + pPhdr[i].p_filesz) <           
                   0; pDyn++) {                                                                     
                if (pDyn->d_tag == DT_STRTAB) {                                                     
                    pOldStrTab = pbyElfFile + pDyn->d_un.d_ptr;                                     
                } else if (pDyn->d_tag == DT_STRSZ) {                                               
                    ulOldStrTabLen = pDyn->d_un.d_val;                                              
                }                                                                                   
            }                                                                                       
            break;                                                                                  
        }                                                                                           
    } //可以用010工具手工查询[图3][图4]0694偏移就是字符串 021b是字符串大小
    
    
    printf("[INFO] get DT_STRTAB success!\n");                                                      
                                                                                                    
    char *pNewStrTab = (char *) (pPhdr + pEhdr->e_phnum);                                           
    uint32_t ulNewStrTabLen = ulOldStrTabLen;                                                                                                                                          
    memcpy(pNewStrTab, pOldStrTab, ulOldStrTabLen); //把字符串表0964指针复制到新的程序头尾部 长度是021b       
    strncpy(pNewStrTab + ulNewStrTabLen, SO_NAME, MAX_NAME_LENGTH); //把libhello.so字符串放到尾部             
    char *pNewModuleName = pNewStrTab + ulNewStrTabLen;                                             
    ulNewStrTabLen += strlen(SO_NAME) + 1;                                                                                                              
    //计算文件尾映射时的虚拟地址                                                                               
    uint32_t ulVAOffset = 0;                                                                        
    uint32_t ulPageSize = sysconf(_SC_PAGESIZE);                                                    
    for (i = 0; i < pEhdr->e_phnum; i++) {                                                          
        if (pPhdr[i].p_type == PT_LOAD) {                                                           
            //由于不能与原始PT_Load项映射的内存地址重叠,                                                         
            //并且原始的PT_Load为了能够映射.bss段而在内存中占用了更多的空间,                                       
            //所以在添加PT_Load项时,在映射的内存地址与文件偏移地址之间需要保留合理的差值,                           
            //即:ulVAOffset变量值                                                                       
            ulVAOffset = pPhdr[i].p_vaddr - pPhdr[i].p_offset;                                      
            ulVAOffset += (pPhdr[i].p_memsz - pPhdr[i].p_filesz) & ~(ulPageSize - 1);               
            ulVAOffset += ulPageSize;                                                               
        }                                                                                           
    }                                                                                               
                                                                                                    
    pPTLoad->p_type = PT_LOAD;                                                                      
    pPTLoad->p_offset = ulFileSize;                                                                 
    pPTLoad->p_vaddr = pPTLoad->p_paddr = pPTLoad->p_offset + ulVAOffset;                           
    pPTLoad->p_filesz = pPTLoad->p_memsz = ulPhdrSize + ulNewStrTabLen;                             
    pPTLoad->p_flags = PF_X | PF_W | PF_R;                                                          
    pPTLoad->p_align = ulPageSize;                                                                  
    printf("[INFO] add PT_LOAD success!\n");                                                        
                                                                                                    
    //修改PT_PHDR中文件偏移和虚拟地址    上面有PT_PHDR函义                                                       
    for (i = 0; i < pEhdr->e_phnum; i++) {                                                          
        if (pPhdr[i].p_type == PT_PHDR) {                                                           
            pPhdr[i].p_offset = ulFileSize;                                                         
            pPhdr[i].p_paddr = pPhdr[i].p_vaddr = ulFileSize + ulVAOffset;                          
            pPhdr[i].p_filesz = pPhdr[i].p_memsz = pEhdr->e_phnum * pEhdr->e_phentsize;             
        }                                                                                           
    }                                                                                               
                                                                                                    
    //修改DT_STRTAB和DT_STRSZ                                                                          
    for (i = 0; i < pEhdr->e_phnum; i++) {                                                          
        if (pPhdr[i].p_type == PT_DYNAMIC) {                                                        
            Elf32_Dyn *pDyn = (Elf32_Dyn *) (pbyElfFile + pPhdr[i].p_offset);                       
            for (; (char *) pDyn - (pbyElfFile + pPhdr[i].p_offset + pPhdr[i].p_filesz) <           
                   0; pDyn++) {                                                                     
                if (pDyn->d_tag == DT_STRTAB) {                                                     
                    pDyn->d_un.d_ptr = pNewStrTab - pbyElfFile + ulVAOffset;                        
                } else if (pDyn->d_tag == DT_STRSZ) {                                               
                    pDyn->d_un.d_val = ulNewStrTabLen;                                              
                } else if (pDyn->d_tag == DT_NULL) {                                                
                    pDyn->d_tag = DT_NEEDED;                                                        
                    pDyn->d_un.d_ptr = pNewModuleName - pNewStrTab;                                 
                    break;                                                                          
                }                                                                                   
            }                                                                                       
            break;                                                                                  
        }                                                                                           
    }                                                                                               
    printf("[INFO] modify DT_STRTAB success!\n");                                                   
    return ulFileSize + ulPhdrSize + ulNewStrTabLen;                                                
}                                                                                                   
```

![图3](https://s2.loli.net/2023/01/11/PJqMhrBCQY3REUK.png)

![图4](https://s2.loli.net/2023/01/11/na2me5CNjUzwhgS.png)

## 参考文献

https://blog.csdn.net/GrayOnDream/article/details/124564129

https://xinqiu.gitbooks.io/linux-inside-zh/content/Theory/linux-theory-2.html

https://paper.seebug.org/papers/Archive/refs/elf/Understanding_ELF.pdf

http://chuquan.me/2018/05/21/elf-introduce/

https://blog.csdn.net/nirendao/article/details/123883856
