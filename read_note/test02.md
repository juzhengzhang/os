# 2. 保护模式内存管理

## 2.1. 内存管理概览
- 分页和分段
  
  分段提供了隔离代码、数据和堆栈模块的机制，使得多个程序（或任务）可以在同一个处理器上运行而不会相互干扰。分页提供了一种实现传统的按需分页的虚拟内存系统的机制，其中程序的执行环境的各个部分根据需要映射到物理内存中。
- 线性地址
  
   处理器的可寻址内存空间，个人理解就是一个个字节。
- 逻辑地址
  
  系统中的所有段都包含在处理器的线性地址空间中。要在特定段中定位一个字节，必须提供一个逻辑地址。逻辑地址由段选择子和偏移量组成。
  
  段选择子是段的唯一标识符。它提供了一个偏移量到描述符表（如全局描述符表，GDT）中的数据结构的位置，称为段描述符。
  
  每个段都有一个段描述符，它指定段的大小、段的访问权限和特权级别、段的类型以及段的第一个字节在线性地址空间中的位置（称为段的基地址）。
  
  逻辑地址的偏移量部分加到段的基地址上，以在段中定位一个字节。因此，基地址加上偏移量形成处理器线性地址空间中的线性地址。

- 物理地址
  
    - 物理地址空间定义为处理器在其地址总线上可以生成的地址范围。 
    - 如果不使用分页，处理器的线性地址空间直接映射到处理器的物理地址空间。
    - 如果使用分页，当一个程序（或任务）试图访问线性地址空间中的一个地址位置时，处理器使用页面目录和页面表将线性地址转换为物理地址，然后对内存位置执行请求的操作（读或写）。 如果正在访问的页面当前不在物理内存中，处理器中断程序的执行（通过产生一个页面错误异常）。操作系统或执行程序然后从磁盘读取页面到物理内存，并继续执行程序。

- 总结
  
    在x86系统中，逻辑地址（Logical Address）是由两部分组成的：段选择符（Segment Selector）和偏移量（Offset）。逻辑地址用于访问段式存储器模型中的内存地址，其中段选择符提供了对段描述符表的索引，该表包含了关于段的信息，如段的起始地址，段的大小，以及段的权限等。偏移量指定了相对于段起始地址的内存地址。

    线性地址（Linear Address）是x86处理器在保护模式下使用的地址空间。它是逻辑地址通过段选择符所指向的段描述符中的基地址加上偏移量得到的结果。线性地址是由操作系统分配给每个程序的虚拟地址空间，程序对该地址空间进行读写操作。在保护模式下，x86处理器可以使用分段机制来隔离各个程序的内存空间。

    物理地址（Physical Address）是指内存中实际存储数据的地址。当程序访问内存时，处理器会把线性地址转换为物理地址。物理地址是通过分页机制得到的，分页机制将虚拟内存映射到物理内存上。

    在x86系统中，逻辑地址通过段选择符和偏移量转换为线性地址，而线性地址通过分页机制转换为物理地址。这些地址之间的转换是通过硬件和操作系统内核完成的。逻辑地址和线性地址的转换使用段描述符表和段选择符寄存器，而线性地址和物理地址的转换使用页表和页目录表。

## 2.2. 分段机制
- Basic Flat Model
  
  Basic Flat Model是一种简单的分段模型，将所有的段都设置成相同的大小和属性，即所有的段都是平等的。在这种模型下，所有的程序都共享同一个段，段的起始地址为0，大小为4GB。这种模型简单易懂，但是无法提供有效的隔离和保护机制。

  至少有两个段描述符，一个用于引用代码段，一个用于引用数据段。但是，这两个段都映射到整个线性地址空间

- Protected Flat Model
  
  Protected Flat Model与Basic Flat Model相似，只是段限制被设置为只包含实际存在的物理内存的地址范围，任何试图访问不存在的内存的操作都会产生异常。
- Multi-Segment Model
  
  它在保护模式下提供了对多个段的支持。在这种模型中，每个程序都有自己的代码段、数据段和堆栈段，这些段都有独立的段选择子和段描述符，使得多个程序可以在同一个系统上运行而不会相互干扰。这种模型可以通过使用任务切换（Task Switching）来实现多任务，并通过使用中断和异常来实现内核操作系统。然而，这种模型存在一些缺点，例如需要频繁地切换段选择子，导致开销较大。
- 总结
  
  在Flat Model中，只有一个代码段、一个数据段和一个堆栈段，所有程序都共享这些段。在Protected Flat Model中，每个程序仍然有自己的代码段、数据段和堆栈段，但是它们的段基地址都设置为0，这样它们就共享相同的物理地址空间。这种模型在多任务环境下更加高效，因为任务切换时不需要切换段选择子。

## 2.3. 逻辑地址和线性地址的转换
- 段选择子 
  
    在x86系统的分段机制中，每个段都有一个唯一的段选择子，用于在全局描述符表（GDT）或局部描述符表（LDT）中定位段描述符。段选择子是一个16位的值，其中包括一个13位的偏移量（指向段描述符表中的条目）和3个特权级位（用于访问控制）。段选择子的格式如下：
    
    |  RPL   | TI/Index  | Table Offset|
    |  ----  | ----  |----|
    | 2bit| 1bit|13bit|
    

    其中RPL为请求特权级，TI（Table Indicator）和Index共同表示使用GDT还是LDT，Table Offset表示描述符表中的偏移量。

    当CPU访问一个地址时，它使用地址中的段选择子来定位对应的段描述符，并从中获取段基址和段限长。然后，CPU使用段基址和地址中的偏移量来计算出物理地址。

    段选择子提供了对访问控制和特权级的支持，它可以控制不同段的访问权限，并且可以防止低特权级的程序访问高特权级的数据。
- 段寄存器 
  
  在x86系统的分段机制中，CPU有6个段寄存器，分别是CS、DS、ES、SS、FS和GS。每个段寄存器都存储了一个段选择子，用于指定当前正在使用的代码段、数据段、堆栈段以及额外的数据段。不同的段寄存器在不同的指令中被用作基址、偏移量或索引寄存器，以计算内存地址。

  段寄存器可以通过MOV指令加载，如下所示：
  ``` 
  MOV segment_register, segment_selector 
  ``` 
  其中segment_register是段寄存器，segment_selector是一个段选择子。当加载一个新的段选择子时，CPU会自动使用新的段选择子来定位对应的段描述符，并从中获取段基址和段限长。

  在实模式下，段寄存器直接指向内存，段寄存器的值被视为一个16位的段地址，加上另一个16位的偏移量，就可以得到实际的物理地址。在保护模式下，段寄存器中存储的是一个段选择子，该选择子指向全局描述符表（GDT）或局部描述符表（LDT）中的一个段描述符，从而可以在访问内存时提供更加灵活的地址映射和访问控制。
- 段描述子
  
  在x86系统中，段描述符（Segment Descriptor）是用于描述每个段的数据结构，它包含了该段的一些属性信息，如段的起始地址、段的长度、段的访问权限等。在操作系统内部，每个进程都有自己的段描述符表（GDT、LDT），用于存储这些描述符，以便于CPU进行地址转换和内存保护。

  以下是x86系统中段描述子（Segment Descriptor）的结构：

  | 位  | 长度 | 描述 |
  |-----|------|------|
  | 0-15| 16   | 段界限15:0 |
  | 16-31| 16   | 段基地址15:0 |
  | 32-39| 8    | 段基地址23:16 |
  | 40-43| 4    | 段类型，包括代码段、数据段、系统段等 |
  | 44-47| 4    | 段描述符类型，包括系统描述符、代码或数据描述符等 |
  | 48-51| 4    | DPL（Descriptor Privilege Level）特权级 |
  | 52   | 1    | 段存在标志 |
  | 53-54| 2    | 段特殊标志，包括代码段的执行属性、数据段的方向等 |
  | 55-63| 9    | 段界限19:16 和 AVL（Available）标志 |
  | 56-63| 8    | 段基地址31:24 |

  段界限（Limit）指定了该段的长度或大小，由于x86架构中的寻址空间为32位，因此段界限最大只能表示2^32-1个字节（即4GB）的内存。

  基地址（Base）指定了该段的起始地址，由于x86架构中的寻址空间为32位，因此基地址最大只能表示2^32-1个字节（即4GB）的内存。

  G：粒度位，表示Limit的单位是字节还是4KB。

  D/B：默认操作数大小或默认栈指针大小和/或上限位。对于代码段，它表示操作数是16位还是32位；对于数据段，它表示栈指针是16位还是32位；对于可扩展的下行栈段，它表示栈段的上限是0xFFFFFFFF还是0x0000FFFF。

  L：64位代码段（IA-32e 模式下有效），表示该段是否为64位代码段。

  AVL：可供软件使用的位，没有特殊含义。

  P：存在位，表示该段是否在内存中。

  DPL：描述符特权级，表示该段的访问权限，从0到3，数字越小权限越高。

  S：描述符类型，表示该描述符是系统描述符还是代码或数据描述符。

  Type：段类型，表示该段的具体属性，如可读、可写、可执行等。

- 总结
 
  在x86系统中，将逻辑地址转换成线性地址的过程称为地址转换（Address Translation）。处理器执行地址转换的过程如下：

  1. 将逻辑地址中的段选择子（Segment Selector）与段描述子表（如GDT或LDT）中的段描述子（Segment Descriptor）进行匹配，以确定该逻辑地址所属的段的起始地址和段限长（Segment Limit）。
  2. 根据段选择子中的索引值，在段描述子表中查找相应的段描述子。
  3. 使用段描述子中的基地址（Base Address）和逻辑地址中的偏移量（Offset）计算出线性地址（Linear Address）。
  线性地址 = 基地址 + 偏移量

  需要注意的是，在分段机制中，逻辑地址是由“段选择子: 偏移量”组成，其中“段选择子”指向段描述子表中的一个段描述子，“偏移量”指向该段内部的某个具体位置。因此，通过段选择子，处理器可以找到该逻辑地址所属的段的信息，然后使用段描述子中的信息计算出线性地址。

  在执行地址转换的过程中，处理器还会进行一些额外的操作，如检查段的特权级（Privilege Level）以确保访问权限的正确性，以及检查段的类型（Code Segment还是Data Segment）以确保执行正确的操作。

## 2.4. 描述符的分类

  在x86系统中，描述符用于描述段或者门的属性。以下是几种不同类型的描述符：

  1. 数据段描述符（Data segment Descriptor）：用于描述数据段的属性，包括段基址、段限长、特权级、段类型等信息。

  2. 代码段描述符（Code segment Descriptor）：用于描述代码段的属性，包括段基址、段限长、特权级、段类型等信息。

  3. 局部描述符表描述符（Local descriptor-table (LDT) segment descriptor）：用于描述LDT的属性，包括LDT的段基址、段限长、特权级等信息。

  4. 任务状态段描述符（Task-state segment (TSS) descriptor）：用于描述任务状态段的属性，包括TSS的段基址、段限长、特权级等信息。

  5. 调用门描述符（Call-gate descriptor）：用于描述调用门的属性，包括目标代码段的选择子、目标偏移量、特权级等信息。

  6. 中断门描述符（Interrupt-gate descriptor）：用于描述中断门的属性，包括目标代码段的选择子、目标偏移量、特权级等信息。

  7. 陷阱门描述符（Trap-gate descriptor）：用于描述陷阱门的属性，包括目标代码段的选择子、目标偏移量、特权级等信息。

  8. 任务门描述符（Task-gate descriptor）：用于描述任务门的属性，包括目标TSS的选择子、特权级等信息。

  这些描述符包含的属性不同，适用于不同的场景。处理器根据逻辑地址中的段选择子，找到对应的段描述符，再根据段描述符中的信息进行地址转换。