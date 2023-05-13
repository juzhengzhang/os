# 1. 调试分析 Linux 0.00 引导程序
## 1.1. 实验目的
- 熟悉实验环境；

- 掌握如何手写Bochs虚拟机的配置文件；

- 掌握Bochs虚拟机的调试技巧；

- 掌握操作系统启动的步骤

## 1.2. 实验报告

  1. 请简述 head.s 的工作原理
   - 重新设置GDT，添加tss0，cs0，ds0，tss1，cs1，ds1等几个描述符
   - 设置IDT，设置好几个必要的中断处理函数
   - 为task0，task1进程设置必要的数据结构：tss段，栈等
   - 实现task0和task1的业务逻辑
   - 实现进程调度逻辑
   - 实现由内核态通过切换到用户态的逻辑



  2. 请记录 head.s 的内存分布状况，写明每个数据段，代码段，栈段的起始与终止的内存地址
   - 首先，我们将断点设至head.s的iret处，此时gdt的初始化已经完成，所以我们可以根据此时gdt的内容初步分析内存分布。
  ![alt text](.\images\1.png)
  gdtr的内容如上图所示，则我们可以查看此处的内存信息，分析gdt。
  ![alt text](.\images\2.png)
  通过上图我们可以获知，第二项为代码段，size为0x7FFFFF ，基地址为0。则实际段长度为8MB。第三项为数据段，size为0x7FFFFF ，基地址为0。则实际段长度为8MB。
  ![alt text](.\images\3.png)
  通过ss的内容，我们可以得知ss指向gdt的第1项，也就是说此时堆栈段同内核数据段。
  <br>
  <br>

     
       
  
  - 然后，我们继续执行程序，此时应进入了任务0，我们查看ldtr，获取ldt的位置信息，查看ldt的内容。ldt的内容如下图所示。
  ![alt text](.\images\4.png)
  根据分析可知，第二项是局部代码段描述符（s=1,type=1010），基地址为0，段限长为0x03ff。
  第三项是局部数据段描述符（s=1,type=0010），基地址为0，段限长为0x03ff。
  <br>
  <br>
  - 然后，我们进入任务1，查看ldt的内容如下图所示。
  ![alt text](.\images\task1_ldt.png)
  同任务1分析，代码段与数据段基地址均为0，段限长为0x03ff，实际长度为4MB。
  <br>
  <br>

  - 综上所述
  初始化的代码段起始地址为0，段长为8MB，则终止地址为0X800000。数据段起始地址为0XB8000，段长为8kb，终止地址为0XBA000。
  任务0的代码段起始地址为0，段长为4MB，则终止地址为0X400000。数据段起始地址为0，段长为4MB，终止地址为0X400000。
  任务1的代码段起始地址为0，段长为4MB，则终止地址为0X400000。数据段起始地址为0，段长为4MB，终止地址为0X400000。
  <br>
  <br>






  3. 简述 head.s 57 至 62 行在做什么？
   我们先看一下IA32手册上对IRET指令的解释：
  IRET指令一一对应地弹出IP指令指针和CS代码段选择符以及EFLAGS的值到EIP，CS和EFLAGS寄存器中，然后继续执行中断的程序。如果返回到另一个特权级，那么这个指令再继续执行前还要弹出栈指针和SS寄存器。
  在此处，我们想要到用户模式执行用户程序，但是我们无法利用jump指令从内核模式跳到
  用户模式，所以使用iret指令，将相应的信息先压入栈中，再通过iret，从而切换到特权级3的任务0中执行。
  具体操作为：
  57：将任务0当前局部空间数据段选择符入栈，
  58：将堆栈指针入栈，
  59：将标志寄存器值入栈，
  60：将当前局部空间代码段选择符入栈，
  61：将代码指针入栈，
  62：通过执行中断返回指令，切换到特权级3的任务0执行。
  <br>
  <br>
    




  4. 简述 iret 执行后， pc 如何找到下一条指令？
   ![alt text](.\images\re_before_iret.png)
   在iret执行前，cs的值为0008，然后执行iret。
   ![alt text](.\images\re_after_iret.png)
   如上图所示，iret执行后，栈中的内容被放至各个相应的寄存器，cs和eip的内容被修改，从而获得了下一条指令的地址。
   <br>
  5. 记录 iret 执行前后，栈是如何变化的？
   执行前栈的情况为：
    ![alt text](.\images\stack_before.png)
    <br>
    执行后栈的情况为：
    ![alt text](.\images\stack_after.png)
    <br>
    可以看出，执行后的栈比执行前的栈正好少5项,并且0xBC4以及0XBC8的内容正好可以与第4题中iret指令执行后的eip，cs对应上，也验证了我们第四题的答案。
    <br>
  6. 当任务进行系统调用时，即 int 0x80 时，记录栈的变化情况。
   调用int 0x80前:
   ![alt text](.\images\int_be.png)
   <br>
   调用int 0x80后:
   ![alt text](.\images\int_af.png)
   由于执行前后任务的特权级不同(从cs可以看出)，所以进行了栈切换。ss寄存器的值发生了相应变化。











![alt text](.\images\bing.jpg)