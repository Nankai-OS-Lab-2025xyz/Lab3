## 练习一

请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写kern/trap/trap.c函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”，在打印完10行后调用sbi.h中的shut_down()函数关机。

要求完成问题1提出的相关函数实现，提交改进后的源代码包（可以编译执行），并在实验报告中简要说明实现过程和定时器中断中断处理的流程。实现要求的部分代码后，运行整个系统，大约每1秒会输出一次”100 ticks”，输出10行。

------

### 相关内容分析

我们打开teap.c中的相关代码，内容，本次实验要求我们是操作系统遇到100次时钟中断之后，向屏幕上打印文字“100 ticks”，首先我们先看一下代码，发现，代码给出了我们的相关打印函数的内容，也给出了我们相应的宏定义的时钟中断100次。

```c
#define TICK_NUM 100

static void print_ticks() {
    cprintf("%d ticks\n", TICK_NUM);
#ifdef DEBUG_GRADE
    cprintf("End of Test.\n");
    panic("EOT: kernel seems ok.");
#endif
}
```

所以我们需要利用这个打印函数来进行我们的调用子程序。

### 代码书写

```c
 static int num = 0;
// "All bits besides SSIP and USIP in the sip register are
            // read-only." -- privileged spec1.9.1, 4.1.4, p59
            // In fact, Call sbi_set_timer will clear STIP, or you can clear it
            // directly.
            // cprintf("Supervisor timer interrupt\n");
             /* LAB3 EXERCISE1   YOUR CODE :  */
            /*(1)设置下次时钟中断- clock_set_next_event()
             *(2)计数器（ticks）加一
             *(3)当计数器加到100的时候，我们会输出一个`100ticks`表示我们触发了100次时钟中断，同时打印次数（num）加一
            * (4)判断打印次数，当打印次数为10时，调用<sbi.h>中的关机函数关机
            */
            clock_set_next_event();//发生这次时钟中断的时候，我们要设置下一次时钟中断
            if (++ticks % TICK_NUM == 0) {
                print_ticks();
                if(++num==10){
                    sbi_shutdown();
                }
            }
```

首先我们根据要求设置*下次时钟中断- clock_set_next_event()*，接下来我们的计时器每一次如果时钟中断需要加一，也就是*++ticks*，此外，当我们的计数器加到100之后，我们就需要进行一个print的操作，进行一个条件判断语句，我们这里使用了宏定义也就是100，每一百次时钟中断我们进行print输出，目前我们的输出解决了，接下来就是我们相关当打印次数为10时，我们调用相关的关机函数进行关机。

我们首先设置一个静态的变量 *static int num = 0*，这个变量用来记录我们的print的次数，当print次数达到10次时，我们就调用关机函数进行关机。

自此，我们的练习一结束，实验结果如下：

![image-20251026165122440](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\image-20251026165122440.png)

## 扩展练习Challenge3：完善异常中断

编程完善在触发一条非法指令异常 mret和，在 kern/trap/trap.c的异常处理函数中捕获，并对其进行处理，简单输出异常类型和异常指令触发地址，即“Illegal instruction caught at 0x(地址)”，“ebreak caught at 0x（地址）”与“Exception type:Illegal instruction"，“Exception type: breakpoint”。

------

### 相关指令异常处理代码补充

首先，我们要清楚`tf->epc` 是结构体 `trapframe`（中断帧）中的一个字段，对应于 CPU 的 `sepc` 寄存器。其意义就是保存异常发生时的指令地址。当发生异常或中断时，RISC-V 硬件会自动把当前正在执行的指令地址保存到 `sepc`。

接下来，由于我们是异常指令会触发异常，所以当我们异常处理结束之后，我们需要将seqc寄存器指向下一条指令，避免一直在异常处理中。

以此我们来补充一下我们的代码。

trap.c文件

```c
case CAUSE_ILLEGAL_INSTRUCTION:
             // 非法指令异常处理
             /* LAB3 CHALLENGE3   YOUR CODE :  */
            /*(1)输出指令异常类型（ Illegal instruction）
             *(2)输出异常指令地址
             *(3)更新 tf->epc寄存器
            */
            cprintf("Exception type: Illegal instruction\n");
            cprintf("Illegal instruction caught at 0x%08x\n", tf->epc);
            tf->epc += 4; // 跳过触发异常的那条指令
            break;
        case CAUSE_BREAKPOINT:
            //断点异常处理
            /* LAB3 CHALLLENGE3   YOUR CODE :  */
            /*(1)输出指令异常类型（ breakpoint）
             *(2)输出异常指令地址
             *(3)更新 tf->epc寄存器
            */
            cprintf("Exception type: breakpoint\n");

            cprintf("ebreak caught at 0x%08x\n", tf->epc);
            tf->epc += 4; // 同样跳过断点指令
            break;
```

ok，我们完成了对于异常处理代码的补充部分。

### 添加相关异常指令

首先我们要知道如何添加相关的异常指令，可以通过内联汇编加入，也可以直接修改汇编。为了我们快速测试我们的异常处理，我们选择使用内联汇编的方式（也就是直接在init.c中写入）。

我们了解到

`ebreak`：标准的 **断点指令**，执行时会产生 **Breakpoint Exception**（`scause = CAUSE_BREAKPOINT`）。用于调试/测试，行为确定。

`mret`：是 **特权返回指令**（从 Machine mode 返回），**在不适当的权限下执行会产生 Illegal Instruction**，所以常被用来触发 **Illegal Instruction** 来做测试（注意它是特权指令，实际效果依执行上下文和特权级而异）。

以此我们来使用内联汇编的方式，来使用这两条指令作为我们添加的异常指令。

`asm`：内联汇编。

`volatile`：告诉编译器“**这条指令有副作用，不能删、不能移走**”。否则在优化时编译器可能会把它折叠/消除，导致测试无效。

```c
int kern_init(void) {
    extern char edata[], end[];
    // 先清零 BSS，再读取并保存 DTB 的内存信息，避免被清零覆盖（为了解释变化 正式上传时我觉得应该删去这句话）
    memset(edata, 0, end - edata);
    dtb_init();
    cons_init();  // init the console
    const char *message = "(THU.CST) os is loading ...\0";
    //cprintf("%s\n\n", message);
    cputs(message);

    print_kerninfo();

    // grade_backtrace();
    idt_init();  // init interrupt descriptor table

    pmm_init();  // init physical memory management

    idt_init();  // init interrupt descriptor table

    clock_init();   // init clock interrupt
    intr_enable();  // enable irq interrupt
// ======== Challenge 3: 测试异常处理 ========
    cprintf("\n[TEST] Triggering Illegal Instruction and Breakpoint...\n");

    // 触发非法指令异常（mret 在内核态不允许执行）
    asm volatile("mret");

    // 触发断点异常（ebreak）
   asm volatile(
    "ebreak\n\t"
    "nop\n\t"
    );
    /* do nothing */
    while (1)
        ;
}
```

我们的异常指令会在 C 程序的“顺序流”中执行，也就是在 `intr_enable()` （开启终端响应能力）之后执行异常指令。

接下来，我们来验证一下实验。

![image-20251026195525041](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\image-20251026195525041.png)

可以发现我们完成了异常处理的逻辑。

#### 一些问题

在实验过程中，我们发现，当我们使用`ebreak`指令之后，进入到了异常处理的程序后，我们明明添加了` tf->epc += 4`;也就是进入到下一条指令，但是结果却是，我们下一条指令是不知名的异常，进入了default的那个分支，后来我在`ebreak`指令之后添加了一个空指令，解决了这个问题，为什么会出现这个问题，查询资料未能找到一些结果，询问助教。