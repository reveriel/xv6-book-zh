# note chap1

allocproc is called for each new process.
- scans proc table for an empty slot.
- allocate a kernel stack
    - 

This should be read with intel's manuel[chap07 multitasking](https://pdos.csail.mit.edu/6.828/2018/readings/i386/c07.htm)

When doing thing close to hardware, It's crucial to know which functionality are provided 
by h/w and which by s/w.

hardware support: 
- task state segment
- task state segment descriptor
- task register
- task gate descriptor

> I had a book talking about intel's cpu about didn't read it, and gave to my roomate who 
> does not major in CS.  [The Intel microprocessors](https://book.douban.com/subject/4872461/)
> Stupid .. I am.

> It's damn too long... that book.
> 也难怪, 当时学这个课只是为了汇编. 只要看一点点就能写汇编程序了. 其实, 不看这书那门课
> 也能过, 后面的 IO bus, 特别是 中断异常多任务功能, 完全没有了解, 没有考也没有用到.
> 这些都是必须和操作系统相结合才能理解的.
> 学校的课程设置真糟糕.

1. Interrupts and exceptions can cause task switchs. auto switch back when finished. and
   Interrupt tasks may interrupt lower-priority interrupt tasks to any depth.
2. when switch, x86 can switch to another LDT(local descriptor table) and to another page directory.


## task state segment, TSS

the fidels of a Tss belong to two clasess:
1. a dynamic set that the processor updates with each switch from the task. includes:
  - general registers (eax, ecx, edx, ebx, esp, ebp, esi, edi)
  - seg regs (es, cs, ss, ds, fs, gs)
  - flags (eflags)
  - eip
  - the selector of the TSS of the previously executing task. (updated only when a
    return is expected)
2. a static set that the processor reads but does not change. includes:
  - selector of the task's LDT
  - PDBR(or CR3), the reg that contains the base address of the task's page directory.
  - pointers to the stasks for privilege levels 0-2
    - ss0:esp0, ss1:esp1, ss2:esp2
  - the T-bit(debug trap bit) which causes the processor to raise a debug exxception when
    a task switch occurs. used for debugging.
  - the I/O map base. (???)

## tss descriptor

This is a descriptor for tss, fields:

- Type 
  - B: whether the tasks is busy. This allows the processor to detect a switch to a task
    that is already busy
- Base(32bit),  the location in 4GB linear space.
- limit(20bit), the size. must >= 103, or causes an exception.
- DPL, descriptor privilege level.

A procedure has access to a tss descriptor can cause a task switch. the DPL should be set
to 0 so that only trusted one can do task switching.

read and write to a tss-descriptor can be accomplished only with another descripotr that
redefineds the tss as a data segment. 

tss descriptors may reside only in the GDT.

## task register

TR(task register) identifies the currently executing task by pointing to the tss.

tr:
  - visible part(16bit), read/writable, selects a tss descirptor in the GDT.
  - invisible part(base + limit), cache the base and limit values from the tss descriptor.

instructions LTR, STR are used to write and read the visible portion of tr.
- ltr: loads tr, must select a tss descriptor in GDT. priviledged 0.
  - may be exected only when CPL is zero. This is generally used during system init to
    give an initial value to the tr, thereafter, tr are changed by task switch ops.
- str: store tr. not privileged.

## task gate descriptor.

a task gate descriptor provides an indirect, protected reference to a TSS.. (yet another
descriptor? A descriptor for descriptor ?): 

yes. a descriptor for task desriptor, but can be in LDT, and IDT.

fields:

- selector(16bit): points to a tss descriptor.
- DPL, the right to use the descriptor to cause a task switch. a procedure may not select
    a task gate descriptor unless the max of the selector's RPL and the CPL of the
    procedure is numerically less that or equal to the DPL of the descriptor. This
    prevents unstrusted procedures from causing a task switch.

a procedure that as access to a task gate has the power to cause a task switch. the 386
has tasks gates in addition to tss to satisfy three needs: 看来是新加的.
1. the need for a task to have a single busy bit.  the busy-bit is stored in the tss
   descriptor, each task should hav only one such descriptor.
2. The need to provide slective access to tasks. tasks gates ban reside in LDTs.
3. The need for an interrupt or exception to cause a task switch. task gates can reside
   in the IDT.

## task switching

switch happens when:
1. current task executes a jmp or call that refers to a tss desccriptor.
2. current task executes a jmp or call that refers to a task gate
3. an interrupt or exception vectors th a task gate in the IDT
4. current task executes an iret when NT(nested task) flag is net.

descriptors have TYPE fields that distinguished between various kinds of them.

if jmp or call refer to a tss descriptor or to a task gate. switche to the indicated task.

exception or interrupt causes a task switch when it vectors to a task gate in the IDT.
If it vectors to an interttupt or trap gate in the IDT, a task switch does not occur.

whether invoked as a task or as a procedure of the interrupted task, an interrupt handler
alwasy returns control to the interrupted procedure in the interrupted task. 
... 过会儿回头再理解.

从这里看, interrupt 有几种形式啊..

task switching steps:
1. check DPL of the target descritpro. compared with CPL, and RPL of the gate
   selector???
   exceptions, interrupts and iret no check.
2. check tss's present, limit.
3. save the state of the current task. the processor finds the base address of the current
   tss cached in the task register. it copies the registers into the current tss. the eip
   fild of the tss points to the instruction after the one that cused the task switch.
4. loding the task register with the selector of the incomming task'ss tss decriptor.
   marking the incoming task's tss descriptor as busy, and set the ts(task switched) bit
   of the msw(machine status word)(about coprocessor)
5. load the incomming task's state from its tss and resuming execution.

这些都是硬件做的.. 不看手册真的看不懂 xv6.


## back link.

the back-link field of the tss and the NT(nested task) bit of the flag word together allow
the 386 to automatitcally return to a task that called another task or was interrupted by
another task.

when (in some condition)switch to a new task, back-link of the new tss is filled, and 
NT flag indicate the back-link field is valid.

interrupt interrupt .... interrrupt, the chain can be long, busy bit prevents loop.







