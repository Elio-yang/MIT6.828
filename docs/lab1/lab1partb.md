# Lab1

## partA 练习

- **exercise3**

  ![image-20211003204855110](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003204855110.png)

  ``gdb``指令：

  ``x/Ni   addr`` :反汇编addr处的N条指令

  ``x/Nx addr``:打印N字节addr处的内存

  ``b *addr``:在addr处设置断点

  readsect(): 0x7c7c

  bootmain():0x7d25

  循环结束的第一条指令是0x7d81处的``call *0x10018``,利用``gdb``，``0x10018``内存处的值为``0x10000c``，故第一条指令是``call 0x10000c``。这个地址就是kernel的entry。

  > At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

  从``ljmp $PROT_MODE_CSEG,$protcseg``这条指令后开始执行32位代码。真正造成切换的，是``CR0``的``PE``位被置为，进入了保护模式。

  > What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?

  last：

  ```assembly
   call *0x10018
  ```

  first：

  ```assembly
  f010000c <entry>:
  f010000c:	66 c7 05 72 04 00 00 	movw   $0x1234,0x472
  ```

  > *Where* is the first instruction of the kernel?

  很显然在0x10000c。

  > How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

  都是通过ELF header得知的。

## Loading the kernel

第一个需要注意的是代码的链接地址和装载地址

使用命令

```shell
objdump -h <file.o>
# -x Display all available header information 
# -f Display entry point
# 更多用法 man objdump 即可
```

![image-20211003210804230](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003210804230.png)

在kernel中这两者是不同的，但是在之前的boot中

![image-20211003210907684](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003210907684.png)

二者是一致的。在``kern/entry.S``中有这样一段代码

```assembly
# Turn on paging.
movl	%cr0, %eax
orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
movl	%eax, %cr0
```

这便开启了地址映射，在此之前kernel的VMA和LMA地址处的内存一般是不同的，但是开启分页之后，LMA映射到了VMA。

## The Kernel

第一个值得注意的是：开启分页模式，将虚拟地址[0, 4MB)映射到物理地址[0, 4MB)，[0xF0000000, 0xF0000000+4MB)映射到[0, 4MB）（/kern/entry.S）

分页模式下的寻址，在Intel手册中也有给出

![image-20211003211930923](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003211930923.png)

开启这个模式的代码如下

```assembly
# Load the physical address of entry_pgdir into cr3.  entry_pgdir
# is defined in entrypgdir.c.
movl	$(RELOC(entry_pgdir)), %eax
movl	%eax, %cr3
# Turn on paging.
movl	%cr0, %eax
orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
movl	%eax, %cr0
```

关于地址的映射在``kern/entrypgdir.c``有代码实现

```c
__attribute__((__aligned__(PGSIZE)))
pde_t entry_pgdir[NPDENTRIES] = {
	// Map VA's [0, 4MB) to PA's [0, 4MB)
	[0]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
	[KERNBASE>>PDXSHIFT]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};
```

编译器分配的空间是强制性4kB页对齐的。``pgdir``是一个1024项的数组。这里可以不用详细了解原理``For now, you don't have to understand the details of how this works, just the effect that it accomplishes.``

- **exercise7**

  ![image-20211003212621242](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003212621242.png)

  在开启分页之前，这两个地址内容是不一致的，后面经过了地址映射，这两者的内容一致。注释掉

  ``movl %eax, %cr0``程序会崩溃。

## Formated Printing to the Console

首先是几个函数的调用关系

![image-20211003213442000](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003213442000.png)

然后练习题

- **exercise8**

  ![image-20211003213520634](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003213520634.png)

   这个文件就是``lib/printfmt.c``

```c
// (unsigned) octal
case 'o':
    // Replace this with your code.
    num=getuint(&ap,lflag);
    base=8;
    goto number;
```

对照上下文很容易补全。

下面是回答问题：

> 1. Explain the interface between `printf.c` and `console.c`. Specifically, what function does `console.c` export? How is this function used by `printf.c`?

对照上文调用关系图即可

> 1. Explain the following from console.c
>
>    ```c
>    if (crt_pos >= CRT_SIZE) {
>        int i;
>        memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) *sizeof(uint16_t));
>        for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
>            crt_buf[i] = 0x0700 | ' ';
>        crt_pos -= CRT_COLS;
>    }
>    ```

首先文本模式最多能显示``25*80``个字符，即25行每行80个。此处

```c
// console.h
#define CRT_ROWS	25
#define CRT_COLS	80
#define CRT_SIZE	(CRT_ROWS * CRT_COLS)
```

因此，这一段处理的是超出一屏幕以后的做法：即舍弃最上面一行，整体上移一行。

![image-20211003214742308](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003214742308.png)

> For the following questions you might wish to consult the notes for Lecture 2. These notes cover GCC's calling convention on the x86.
>
> Trace the execution of the following code step-by-step:
>
> ```c
> int x = 1, y = 3, z = 4;
> cprintf("x %d, y %x, z %d\n", x, y, z);
> ```
>
> - In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?
> - List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.

GCC 函数调用约定是参数从右往左入栈。此处``fmt``指向的就是第一个参数的位置。而``ap``指向第一个可变参数，也就是第二个参数``x``的位置。关于变参数，JOS使用的是GCC builtin来实现的。其实现可以用如下代码进行大致说明（不是严谨的完整实现）：

```c
#define va_start(list,param_1st)   ( list = (va_list)&param1+ sizeof(param_1st) )
#define va_arg(list,type)   ( (type *) ( list += sizeof(type) ) )[-1]
#define va_end(list) ( list = (va_list)0 )
```

因此：

``va_list``：即``char*``

``va_start``:获取第一个可变参数的地址

``va_arg``：返回指向下一个参数的指针

``va_end``：情况参数列表

> 1. Run the following code.
>
>    ```
>        unsigned int i = 0x00646c72;
>        cprintf("H%x Wo%s", 57616, &i);
>    ```
>
>    What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise.
>
>    Here's [an ASCII table](http://web.cs.mun.ca/~michael/c/ascii-table.html)that maps bytes to characters.
>
>    The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?
>
>    [Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](http://www.networksorcery.com/enp/ien/ien137.txt).

把这段代码加入``init.c``中，运行``make qemu``,结果如下

![image-20211003220014708](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003220014708.png)

``0xe110=57616``这很好解释，查阅ASCII表，得知

```
00(\0) 64(d) 6c(l) 72(r)
```

显然这是由于小端模式而使用的一个数。为了证明这一点，可以输出``&i``内存处的字节。将下面这段代码放在上面打印代码的后面

```c
cprintf("addr of i: %p\n",&i);
char *p=(char*)&i;
for(int i=0;i<4;i++){
	cprintf("[%x]",*p);
	p++;
}
```

输出结果如下：

![image-20211003220358001](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003220358001.png)

> In the following code, what is going to be printed after``'y='``? (note: the answer is not a specific value.) Why does this happen?
>
> ```
>     cprintf("x=%d y=%d", 3);
> ```

运行结果如下

![image-20211003220558086](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003220558086.png)

显然y的值并不一定固定，他就是把内存中那个位置的数拿来充当了第二个参数。

> Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?

更改了入栈方式，相应地更改``va_start``和``va_start``即可。

## The Stack

先看这个练习

- **exercise9**

  ![image-20211003220907333](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003220907333.png)

  在``entry.S``中可以找到如下代码

  ```assembly
  # where the stack is set.
  # Clear the frame pointer register (EBP)
  # so that once we get into debugging C code,
  # stack backtraces will be terminated properly.
  movl	$0x0,%ebp			# nuke frame pointer
  # Set the stack pointer
  movl	$(bootstacktop),%esp
  # now to C code
  call	i386_init
  ```

  利用``gdb``得知，``movl	$(bootstacktop),%esp``会被编译为``movl $0xf0110000,%esp``。因此栈何时初始化，栈放在哪儿都清楚了。继续看代码

  ```assembly
  ###################################################################
  # boot stack
  ###################################################################
  	.p2align	PGSHIFT		# force page alignment
  	.globl		bootstack
  bootstack:
  	.space		KSTKSIZE
  	.globl		bootstacktop   
  ```

  这便开辟了栈的大小，即``32KB``。栈由高地址向低地址增长。

  下面，关于函数的调用过程，做一个总结，可以参考[CSAPP，p164]。

  这是从课件ppt截取的两页

  ![image-20211003221514138](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003221514138.png)

  ![image-20211003221628158](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003221628158.png)

  关于函数的调用，一般有如下的动作发生：

  1. **函数调用者(caller)**将参数入栈，按照**从右到左**的顺序入栈
  2. call指令会**自动**将当前**%eip**(指向call的后面一条指令)入栈，ret指令将**自动**从栈中弹出该值到eip寄存器
  3. **被调用函数(callee)**负责：将%ebp入栈，%esp的值赋给%ebp。

  因此函数开头都会是类似的两条指令

  ```assembly
  push %ebp
  mov %esp,%ebp
  ```

  因此整个**调用链**差不多可以描述成如下形式

  ![image-20211003223302656](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003223302656.png)

  来到下一个练习

- **exercise10**

    ![image-20211003223418781](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003223418781.png)

    每次call之后会干什么，上文已经分析了。至于每次递归入栈的字，伪代码可以表示为

    ```assembly
    push %eip
    push %ebp
    push %esi
    push %ebx
    ```

    共计``0x10``B。
    
- **exercise11**

    ![image-20211003223745665](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003223745665.png)

    需要我们更改``mom_backtrace()``函数，达到的效果如下：

    ![image-20211003223847009](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003223847009.png)

    题目中已经说明，获得``%ebp``的函数就是``read_ebp()``。那么编码工作应该很好完成了(利用调用链中``%ebp``的链)

    ```c
    int
    mon_backtrace(int argc, char **argv, struct Trapframe *tf)
    {
    	// Your code here.
    	uint32_t *ebp=(uint32_t*)read_ebp();
    	while(ebp!=NULL){
    	    cprintf("ebp %8x  eip %8x  args %08x %08x %08x %08x 08x\n",
    			ebp,ebp[1],ebp[2],ebp[3],ebp[4],ebp[5],ebp[6]);
    	    ebp=(uint32_t *)(*ebp);
    	}
    	return 0;
    }
    
    ```

    运行结果如下

    ![image-20211003224235544](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003224235544.png)

- **exercise12**

    ![image-20211003224412343](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003224412343.png)

    练习12的任务有三个：

    1. 搞清楚``__STAB_*``
    2. 添加命令``backtrace``
    3. 完善``mon_backtrace``

    - 任务一

      根据提示，查看这几个文件，首先是``kernel.ld``。

      ```
      .stab : {
                      PROVIDE(__STAB_BEGIN__ = .);
                      *(.stab);
                      PROVIDE(__STAB_END__ = .);
                      BYTE(0)         /* Force the linker to allocate space
                                         for this section */
              }
       
      .stabstr : {
                      PROVIDE(__STABSTR_BEGIN__ = .);
                      *(.stabstr);
                      PROVIDE(__STABSTR_END__ = .);
                      BYTE(0)         /* Force the linker to allocate space
                                         for this section */
              }
      ```

      可以知道``.stab``和``.stabstr``应该是两个段。

      接着 ``objdump -h obj/kern/kernel``

      ![image-20211003224905040](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003224905040.png)

      然后是``objdump -G obj/kern/kernel``

      ![image-20211003225158686](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003225158686.png)

      执行后面的操作以后，大致可以知道这是一个段，包含了调试信息。细节可以不用太了解。接着找到``stab.h``，其中

      ![image-20211003225336352](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003225336352.png)

      这两项便是后文编码寻找行号时需要的。下面开始任务二和三

    - 任务二

      题目中提示了需要使用``debuginfo_eip``,查找这个函数发现，他会将需要的信息存到类型为``struct Eipdebuginfo``的结构体中。查看该结构体定义(kern/kdebebug.h)

      ```c
      // Debug information about a particular instruction pointer
      struct Eipdebuginfo {
      	const char *eip_file;		// Source code filename for EIP
      	int eip_line;			    // Source code linenumber for EIP
      
      	const char *eip_fn_name;	// Name of function containing EIP
      					            //  - Note: not null terminated!
          
      	int eip_fn_namelen;		// Length of function name
      	uintptr_t eip_fn_addr;		// Address of start of function
      	int eip_fn_narg;		// Number of function arguments
      };
      ```

      因此只需要使用``debuginfo_eip``填充该结构体，再输出信息即可。

      ```c
      static struct Command commands[] = {
      	{ "help", "Display this list of commands", mon_help },
      	{ "kerninfo", "Display information about the kernel", mon_kerninfo },
      	{ "backtrace", "Show stack backtrace",mon_stacktrace}	
      };
      //......
      int 
      for_stack(int argc,char **argv,struct Trapframe *tf)
      {
      	uint32_t *ebp=(uint32_t*)read_ebp();
      	while(ebp!=NULL){
      	    struct Eipdebuginfo info;
      	    uint32_t eip = ebp[1];
      	    debuginfo_eip((int)eip, &info);
      	    cprintf("  ebp %8x  eip %8x  args %08x %08x %08x %08x 08x\n",
      			ebp,ebp[1],ebp[2],ebp[3],ebp[4],ebp[5],ebp[6]);
      	    const  char* filename=(&info)->eip_file;
      	    int line = (&info)->eip_line;
      	    const char * not_null_ter_fname=(&info)->eip_fn_name;
      	    int offset = (int)(eip)-(int)((&info)->eip_fn_addr);
      	    cprintf("        %s:%d:  %.*s+%d\n",filename,line,info.eip_fn_namelen,not_null_ter_fname,offset);
      	    ebp=(uint32_t *)(*ebp);
      	}
      	return 0;
      }
      int
      mon_stacktrace(int argc,char **argv,struct Trapframe *tf)
      {
          cprintf("Stack backtrace:\n");
          return for_stack(argc,argv,tf);
      }
      ```

      其中关于文件行号的查找实现，对照上下文就能实现，注意``N_SLINE``这就是之前说``stab``时提到的一个有用的属性。

      ```c
      // Search within [lline, rline] for the line number stab.
      // If found, set info->eip_line to the right line number.
      // If not found, return -1.
      //
      // Hint:
      //	There's a particular stabs type used for line numbers.
      //	Look at the STABS documentation and <inc/stab.h> to find
      //	which one.
      // Your code here.
      stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
      if(lline<=rline){
          info->eip_line=stabs[lline].n_desc;
      }else{
          return -1;
      }
      ```

      运行结果如下：

      ![image-20211003230512500](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003230512500.png)

      之后运行评分程序

      ![image-20211003230440507](C:\Users\ELIO\AppData\Roaming\Typora\typora-user-images\image-20211003230440507.png)

      

至此，``Lab1``完结。完整代码仓库可点击：https://github.com/Elio-yang/MIT6.828

