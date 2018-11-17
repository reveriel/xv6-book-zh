## Kernel organization

<!--
A key design question is what part of the operating system should run in kernel mode. One possibility is that the entire operating system resides in the kernel, so that the implementations of all system calls run in kernel mode. This organization is called a monolithic kernel.
-->

一个关键的设计问题是操作系统的哪个部分应该在内核模式下运行。 一种可能性是整个操作系统驻留在内核中，因此所有系统调用的实现都以内核模式运行。 该组织称为宏内核。

<!--
In this organization the entire operating system runs with full hardware privilege. This organization is convenient because the OS designer doesn’t have to decide which part of the operating system doesn’t need full hardware privilege. Furthermore, it easy for diﬀerent parts of the operating system to cooperate. For example, an operating system might have a buﬀer cache that can be shared both by the ﬁle system and the virtual memory system.
-->

在此组织方式中，整个操作系统以完全硬件权限运行。 这个
组织方式很方便，因为OS 设计师不必决定哪个部分操作系统不需要完整的硬件权限。 此外，它很容易适用于不同的部分
操作系统合作。 例如，操作系统可能具有缓存
可以由文件系统和虚拟内存系统共享。

<!--
A downside of the monolithic organization is that the interfaces between different parts of the operating system are often complex (as we will see in the rest of this text), and therefore it is easy for an operating system developer to make a mistake. In a monolithic kernel, a mistake is fatal, because an error in kernel mode will often result in the kernel to fail. If the kernel fails, the computer stops working, and thus all applications fail too. The computer must reboot to start again.
-->

宏内核的一个缺点是，不同部分之间的接口
操作系统通常很复杂（我们将在本文的其余部分中看到），因此它使
操作系统开发人员容易犯错误。 在宏内核中，错误是
致命的，因为内核模式中的错误通常会导致内核失败。 如果内核失败，
计算机停止工作，因此所有应用程序也都会失败。 计算机必须重新启动才能启动
再次。

<!--
To reduce the risk of mistakes in the kernel, OS designers can minimize the amount of operating system code that runs in kernel mode, and execute the bulk of the operating system in user mode. This kernel organization is called a microkernel.
-->

为了降低内核中出错的风险，OS设计人员可以最大限度地减少以内核模式运行的操作
系统代码，并以用户模式执行大部分操作系统。这个内核组织称为微内核。

<!--
Figure 1-1 illustrates this microkernel design. In the ﬁgure, the ﬁle system runs as a user-level process. OS services running as processes are called servers. To allow applications to interact with the ﬁle server, the kernel provides an inter-process communication mechanism to send messages from one user-mode process to another. For example, if an application like the shell wants to read or write a file, it sends a mesage to the file server and waits for a response.
-->

图1-1说明了这种微内核设计。 在图中，文件系统以用户级别运行
处理。 作为进程运行的OS服务称为服务器。 允许应用程序与之交互
在文件服务器中，内核提供了一个进程间通信机制来发送消息
一个用户模式进程到另一个。 例如，如果像shell这样的应用程序想要读取或
写一个文件，它将一个消息发送到文件服务器并等待响应。

Figure 1-1. A microkernel with a ﬁle system server

<!--
In a microkernel, the kernel interface consists of a few low-level functions for starting applications, sending messages, accessing device hardware, etc. This organization allows the kernel to be relatively simple, as most of the operating system resides in user-level servers.
-->

在微内核中，内核接口由一些低级函数组成，用于启动应用程序，发送消息，访问设备硬件等。这种组织允许内核相对简单，因为大多数操作系统都驻留在用户级服务器中。

<!--
Xv6 is implemented as a monolithic kernel, following most Unix operating systems. Thus, in xv6, the kernel interface corresponds to the operating system interface, and the kernel implements the complete operating system. Since xv6 doesn’t provide many services, its kernel is smaller than some microkernels.
-->

Xv6是作为宏内核实现的，遵循大多数Unix操作系统。
因此，在xv6中，内核接口对应于操作系统接口，内核实现完整的操作系统。
由于xv6不提供许多服务，因此其内核比某些微内核要小。


## Process overview

<!--
The unit of isolation in xv6 (as in other Unix operating systems) is a process. The
process abstraction prevents one process from wrecking or spying on another process’s
memory, CPU, file descriptors, etc. It also prevents a process from wrecking the kernel
itself, so that a process can’t subvert the kernel’s isolation mechanisms. The kernel must
implement the process abstraction with care because a buggy or malicious application may
trick the kernel or hardware in doing something bad (e.g., circumventing enforced
isolation). The mechanisms used by the kernel to implement processes include the
user/kernel mode flag, address spaces, and time-slicing of threads.
-->

xv6中的隔离单元（与其他Unix操作系统一样）是一个过程。
进程抽象可以防止一个进程破坏或监视另一个进程的内存，CPU，文件描述符等。它还可以防止进程破坏内核本身，从而使进程无法破坏内核的隔离机制。
内核必须小心地实现进程抽象，因为错误或恶意应用程序可能会欺骗内核或硬件做坏事（例如，规避强制隔离）。
内核用于实现进程的机制包括用户/内核模式标志，地址空间和线程的时间分片。

<!--
To help enforce isolation, the process abstraction provides the illusion to a program that
it has its own private machine. A process provides a program with what appears to be a
private memory system, or address space, which other processes cannot read or write. A
process also provides the program with what appears to be its own CPU to execute the
program’s instructions.
-->

为了帮助实现隔离，进程抽象为程序提供了幻觉，它拥有自己的私有机器。
进程为程序提供看似私有的内存系统或地址空间，其他进程无法读取或写入。
进程还为程序提供了看似自己的CPU来执行程序的指令。

<!--
Xv6 uses page tables (which are implemented by hardware) to give each process its own
address space. The x86 page table translates (or ‘‘maps’’) a virtual address (the address
that an x86 instruction manipulates) to a physical address (an address that the processor
chip sends to main memory).
-->

Xv6使用页表（由硬件实现）为每个进程提供自己的地址空间。
x86页表将虚拟地址（x86指令操作的地址）转换（或“映射”）到物理地址（处理器芯片发送到主存储器的地址）。

<!--
Xv6 maintains a separate page table for each process that deﬁnes that process’s address
space. As illustrated in Figure 1-2, an address space includes the process’s user memory
starting at virtual address zero. Instructions come ﬁrst, followed by global variables,
then the stack, and ﬁnally a ‘‘heap’’ area (for malloc) that the process can expand as
needed.
-->

Xv6为每个进程维护一个单独的页表，用于定义该进程的地址空间。
如图1-2所示，地址空间包括从虚拟地址零开始的进程的用户存储器。
首先是指令，然后是全局变量，然后是堆栈，最后是一个“堆”区域（对于malloc），进程可以根据需要进行扩展。

Figure 1-2 Layout of a virtual address space


<!--
Each process’s address space maps the kernel’s instructions and data as well as the user program’s memory. When a process invokes a system call, the system call executes in the kernel mappings of the process’s address space. This arrangement exists so that the kernel’s system call code can directly refer to user memory. In order to leave plenty of room for user memory, xv6’s address spaces map the kernel at high addresses, starting at 0x80100000.
-->

每个进程的地址空间都映射内核的指令和数据以及用户程序的内存。 当进程调用系统调用时，系统调用将在进程的地址空间的内核映射中执行。 存在这种安排，以便内核的系统调用代码可以直接引用用户内存。 为了给用户内存留出足够的空间，xv6的地址空间在高地址处映射内核，从0x80100000开始。

<!--
The xv6 kernel maintains many pieces of state for each process, which it gathers into a struct proc (2337). A process’s most important pieces of kernel state are its page table, its kernel stack, and its run state. We’ll use the notation p->xxx to refer to elements of the proc structure.
-->

xv6内核为每个进程维护许多状态，并将其收集到struct proc（2337）中。 进程最重要的内核状态是它的页表，内核堆栈和运行状态。 我们将使用符号p-> xxx来引用proc结构的元素。

<!--
Each process has a thread of execution (or thread for short) that executes the process’s instructions. A thread can be suspended and later resumed. To switch transparently between processes, the kernel suspends the currently running thread and resumes another process’s thread. Much of the state of a thread (local variables, function call return addresses) is stored on the thread’s stacks. Each process has two stacks: a user stack and a kernel stack (p->kstack). When the process is executing user instructions, only its user stack is in use, and its kernel stack is empty. When the process enters the kernel (for a system call or interrupt), the kernel code executes on the process’s kernel stack; while a process is in the kernel, its user stack still contains saved data, but isn’t actively used. A process’s thread alternates between actively using its user stack and its kernel stack. The kernel stack is separate (and protected from user code) so that the kernel can execute even if a process has wrecked its user stack.
-->

每个进程都有一个执行线程（或简称线程），它执行进程的指令。线程可以暂停并稍后恢复。为了在进程之间透明地切换，内核暂停当前运行的线程并恢复另一个进程的线程。线程的大部分状态（局部变量，函数调用返回地址）都存储在线程的堆栈中。每个进程都有两个堆栈：用户堆栈和内核堆栈（p-> kstack）。当进程正在执行用户指令时，只有其用户堆栈正在使用，并且其内核堆栈为空。当进程进入内核（由于系统调用或中断）时，内核代码在进程的内核堆栈上执行;当进程在内核中时，其用户堆栈仍包含已保存的数据，但未被主动使用。进程的线程在主动使用其用户堆栈及其内核堆栈之间交替。内核堆栈是独立的（并且不受用户代码保护），因此即使进程已破坏其用户堆栈，内核也可以执行。


<!--
When a process makes a system call, the processor switches to the kernel stack, raises the hardware privilege level, and starts executing the kernel instructions that implement the system call. When the system call completes, the kernel returns to user space: the hardware lowers its privilege level, switches back to the user stack, and resumes executing user instructions just after the system call instruction. A process’s thread can ‘‘block’’ in the kernel to wait for I/O, and resume where it left off when the I/O has finished.
-->

当进程进行系统调用时，处理器切换到内核堆栈，提高硬件权限级别，并开始执行实现系统调用的内核指令。 当系统调用完成时，内核返回用户空间：硬件降低其权限级别，切换回用户堆栈，并在系统调用指令之后继续执行用户指令。 进程的线程可以在内核中“阻塞”以等待I / O，并在I / O完成时从中断处继续。

Figure 1-3. Layout of a virtual address space


<!--
p->state indicates whether the process is allocated, ready to run, running, waiting for I/O, or exiting.
-->

p->state 指示进程状态是刚被分配，准备运行，运行，等待 I/O 还是退出中。

<!--
p->pgdir holds the process’s page table, in the format that the x86 hardware expects. xv6 causes the paging hardware to use a process’s p->pgdir when executing that process. A process’s page table also serves as the record of the addresses of the physical pages allocated to store the process’s memory.
-->

p->pgdir 以x86硬件期望的格式保存进程的页表。 xv6导致分页硬件在执行该进程时使用进程的 p->pgdir。 进程的页表还用作分配用于存储进程内存的物理页的地址记录。


## Code: the first address space

<!--
To make the xv6 organization more concrete, we’ll look how the kernel creates the ﬁrst address space (for itself), how the kernel creates and starts the ﬁrst process, and how that process performs the ﬁrst system call. By tracing these operations we see in detail how xv6 provides strong isolation for processes. The ﬁrst step in providing strong isolation is setting up the kernel to run in its own address space.
-->

为了使xv6组织更具体，我们将看看内核如何创建第一个地址空间（为自己），内核如何创建和启动第一个进程，以及该进程如何执行第一个系统调用。 通过跟踪这些操作，我们详细了解xv6如何为进程提供强大的隔离。 提供强隔离的第一步是将内核设置为在自己的地址空间中运行。

When a PC powers on, it initializes itself and then loads a boot loader from disk into memory and executes it. Appendix B explains the details. Xv6’s boot loader loads the xv6 kernel from disk and executes it starting at entry (1044). The x86 paging hardware is not enabled when the kernel starts; virtual addresses map directly to physical addresses.

当PC启动时，它会自行初始化，然后将启动加载程序从磁盘加载到内存中并执行它。 附录B解释了细节。 Xv6的引导加载程序从磁盘加载xv6内核并从 `entry`（1044）开始执行它。 内核启动时，x86分页硬件未启用; 虚拟地址直接映射到物理地址。

The boot loader loads the xv6 kernel into memory at physical address 0x100000. The reason it doesn’t load the kernel at 0x80100000, where the kernel expects to ﬁnd its instructions and data, is that there may not be any physical memory at such a high address on a small machine. The reason it places the kernel at 0x100000 rather than 0x0 is because the address range 0xa0000:0x100000 contains I/O devices.

引导加载程序将xv6内核加载到物理地址0x100000的内存中。 它没有在内核期望找到其指令和数据的0x80100000加载内核的原因是在小型机器上的这么高的地址上可能没有任何物理内存。 它将内核置于0x100000而不是0x0的原因是因为地址范围0xa0000：0x100000包含I / O设备。

To allow the rest of the kernel to run, entry sets up a page table that maps virtual
addresses starting at 0x80000000 (called KERNBASE (0207)) to physical addresses starting at 0x0 (see Figure 1-2). Setting up two ranges of virtual addresses that map to the same physical memory range is a common use of page tables, and we will see more  examples like this one.

为了允许内核的其余部分运行，条目设置一个页表，将从0x80000000开始的虚拟地址（称为KERNBASE（0207））映射到从0x0开始的物理地址（参见图1-2）。 设置映射到相同物理内存范围的两个虚拟地址范围是页表的常见用法，我们将看到更多这样的示例。

The entry page table is defined in main.c (1306). We look at the details of page tables in Chapter 2, but the short story is that entry 0 maps virtual addresses main+code 0:0x400000 to physical addresses 0:0x400000. This mapping is required as long as entry is executing at low addresses, but will eventually be removed.

入口页表在main.c中定义（1306）。 我们在第2章中查看页表的详细信息，但简短的说明是条目0将虚拟地址main + code 0：0x400000映射到物理地址0：0x400000。 只要`entry`在低地址执行则需要此映射，但最终将被删除，。

Entry 512 maps virtual addresses KERNBASE:KERNBASE+0x400000 to physical addresses 0:0x400000. This entry will be used by the kernel after entry has ﬁnished; it maps the high virtual addresses at which the kernel expects to ﬁnd its instructions and data to the low physical addresses where the boot loader loaded them. This mapping restricts the kernel instructions and data to 4 Mbytes.

条目512将虚拟地址ERNBASE：KERNBASE + 0x400000映射到物理地址0：0x400000。 输入完成后，内核将使用此条目; 它将内核期望发现其指令和数据的高虚拟地址映射到引导加载程序加载它们的低物理地址。 此映射将内核指令和数据限制为4 MB。

Returning to entry, it loads the physical address of entrypgdir into control register %cr3. The value in %cr3 must be a physical address. It wouldn’t make sense for %cr3 to hold the virtual address of entrypgdir, because the paging hardware doesn’t know how to translate virtual addresses yet; it doesn’t have a page table yet. The symbol entrypgdir refers to an address in high memory, and the macro V2P_WO (0213) subtracts KERNBASE in order to find the physical address. To enable the paging hardware, xv6 sets the flag CR0_PG in the control register %cr0.

返回 `entry`，它将 `entrypgdir` 的物理地址加载到控制寄存器％cr3中。 ％cr3中的值必须是物理地址。 ％cr3保留entrypgdir的虚拟地址是没有意义的，因为寻呼硬件还不知道如何翻译虚拟地址; 它还没有页面表。 符号entrypgdir是指高内存中的地址，宏V2P_WO（0213）减去KERNBASE以便找到物理地址。 要启用分页硬件， xv6会在控制寄存器％cr0中设置标志CR0_PG。

The processor is still executing instructions at low addresses after paging is enabled,
which works since entrypgdir maps low addresses. If xv6 had omitted entry 0 from
entrypgdir, the computer would have crashed when trying to execute the instruction after
the one that enabled paging.

在启用分页后，处理器仍然在低地址执行指令，这是因为entrypgdir映射了低地址。
如果xv6从entrypgdir中省略了条目0，则在尝试执行启用分页的指令后，计算机将崩溃。

Now entry needs to transfer to the kernel’s C code, and run it in high memory. First it
makes the stack pointer, %esp, point to memory to be used as a stack (1058). All symbols
have high addresses, including stack, so the stack will still be valid even when the low
mappings are removed. Finally entry jumps to main, which is also a high address. The
indirect jump is needed because the assembler would otherwise generate a PC-relative
direct jump, which would execute the low-memory version of main. Main cannot return, since
the there’s no return PC on the stack. Now the kernel is running in high addresses in the
function main (1217).

现在，入口需要转移到内核的C代码，并在高内存中运行。
首先，它使堆栈指针％esp指向要用作堆栈的内存（1058）。
所有符号都是高地址，包括堆栈，因此即使删除了低映射，堆栈仍然有效。
最后输入跳转到main，这也是一个高地址。
需要间接跳转，因为汇编器否则会生成PC相对直接跳转，这将执行main的低内存版本。
Main无法返回，因为堆栈上没有返回PC。 现在内核在函数main中运行在高地址中（1217）。

## Code: creating the first process

Now we’ll look at how the kernel creates user-level processes and ensures that they are
strongly isolated.

现在我们将看看内核如何创建用户级进程并确保它们是高度隔离的。

Figure 1-4. A new kernel stack

After main (1217) initializes several devices and subsystems, it creates the first process
by calling userinit (2520). Userinit’s first action is to call allocproc. The job of
allocproc (2473) is to allocate a slot (a struct proc) in the process table and to
initialize the parts of the process’s state required for its kernel thread to execute.
Allocproc is called for each new process, while userinit is called only for the very first
process. Allocproc scans the proc table for a slot with state UNUSED (2480-2482). When it
ﬁnds an unused slot, allocproc sets the state to EMBRYO to mark it as used and gives the
process a unique pid (2469-2489). Next, it tries to allocate a kernel stack for the
process's kernel thread. If the memory allocation fails, allocproc changes the
state back to UNUSED and returns zero to signal failure.

在main（1217）初始化几个设备和子系统之后，它通过调用userinit（2520）创建第一个进程。
Userinit的第一个动作是调用allocproc。
allocproc（2473）的工作是在进程表中分配一个槽（struct
proc）并初始化其内核线程执行所需的进程状态部分。
为每个新进程调用Allocproc，而仅为第一个进程调用userinit。
Allocproc扫描proc表以查找状态为UNUSED（2480-2482）的插槽。
当它找到一个未使用的槽时，allocproc将状态设置为EMBRYO以将其标记为已使用，并为进程提供唯一的pid（2469-2489）。
接下来，它尝试为进程的内核线程分配内核堆栈。
如果内存分配失败，allocproc改变状态恢复到未使用，并返回零表示故障。

Now allocproc must set up the new process's kernel stack. allocproc is written
so that it can be used by fork as well as when creating the first process. allocproc
sets up the new process with a specially prepared kernel stack and set of kernel rigsters
that cause it to "return" to user space when if first runs. The layout of the prepared
kernel stack will be as shown in Figure 1-4. allocproc does part of this work by setting
up return program counter values that will cause the new process’s kernel thread to ﬁrst
execute in forkret and then in trapret (2507-2512). The kernel thread will start executing
with register contents copied from p->context. Thus setting p->context->eip to forkret will
cause the kernel thread to execute at the start of forkret (2853). This function will
return to whatever address is at the bottom of the stack. The context switch code (3059)
sets the stack pointer to point just beyond the end of p->context. allocproc places
p->context on the stack, and puts a pointer to trapret just above it; that is where
forkret will return. trapret restores user registers from values stored at the top of the
kernel stack and jumps into the process (3324). This setup is the same for ordinary fork
and for creating the ﬁrst process, though in the latter case the process will start
executing at user-space location zero rather than at a return from fork.

现在allocproc必须设置新进程的内核堆栈。
allocproc，在fork和创建第一个进程时都可以使用它。
allocproc使用专门准备的内核堆栈和一组内核寄存器来设置新进程，这使得它在首次运行时“返回”用户空间。
准备好的内核堆栈的布局如图1-4所示。
allocproc通过设置返回程序计数器值来完成这项工作的一部分，这将导致新进程的内核线程首先在forkret中执行，然后在trapret中执行（2507-2512）。
内核线程将从p->context复制的寄存器内容开始执行。 因此，将p-> context->
eip设置为forkret将导致内核线程在forkret的开头执行（2853）。



