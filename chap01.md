## Kernel organization

<!--
A key design question is what part of the operating system should run in kernel mode. One possibility is that the entire operating system resides in the kernel, so that the implementations of all system calls run in kernel mode. This organization is called a monolithic kernel.
-->

一个关键的设计问题是操作系统的哪个部分应该在内核模式下运行。 一种可能性是整个操作系统驻留在内核中，因此所有系统调用的实现都以内核模式运行。 该组织称为宏内核。

<!--
In this organization the entire operating system runs with full hardware privilege. This organization is convenient because the OS designer doesn’t have to decide which part of the operating system doesn’t need full hardware privilege. Furthermore, it easy for diﬀerent parts of the operating system to cooperate. For example, an operating system might have a buﬀer cache that can be shared both by the ﬁle system and the virtual memory system.
-->

在此组织方式中，整个操作系统以完全硬件权限运行。
这个组织方式很方便，因为OS 设计者不必决定哪个部分操作系统不需要完整的硬件权限。
此外，它很容易适用于不同的部分操作系统合作。
例如，操作系统可能具有可以由文件系统和虚拟内存系统共享的缓存。

<!--
A downside of the monolithic organization is that the interfaces between different parts of the operating system are often complex (as we will see in the rest of this text), and therefore it is easy for an operating system developer to make a mistake. In a monolithic kernel, a mistake is fatal, because an error in kernel mode will often result in the kernel to fail. If the kernel fails, the computer stops working, and thus all applications fail too. The computer must reboot to start again.
-->

宏内核的一个缺点是，不同部分之间的接口操作系统通常很复杂
（我们将在本文的其余部分中看到），因此它使操作系统开发人员容易犯错误。
在宏内核中，错误是致命的，因为内核模式中的错误通常会导致内核失败。
如果内核失败，计算机停止工作，因此所有应用程序也都会失败。 
计算机必须重新启动才能再次启动 。

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

为了帮助实现隔离，进程抽象为程序提供了幻觉，让它认为私有机器。
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

Xv6 为每个进程维护一个单独的页表，用于定义该进程的地址空间。
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

xv6 内核为每个进程维护许多状态，并将其收集到`struct proc`（2337）中。 进程最重要的内核状态是它的页表，内核堆栈和运行状态。 我们将使用符号`p->xxx` 
来引用 `proc` 结构的元素。

<!--
Each process has a thread of execution (or thread for short) that executes the process’s instructions. A thread can be suspended and later resumed. To switch transparently between processes, the kernel suspends the currently running thread and resumes another process’s thread. Much of the state of a thread (local variables, function call return addresses) is stored on the thread’s stacks. Each process has two stacks: a user stack and a kernel stack (p->kstack). When the process is executing user instructions, only its user stack is in use, and its kernel stack is empty. When the process enters the kernel (for a system call or interrupt), the kernel code executes on the process’s kernel stack; while a process is in the kernel, its user stack still contains saved data, but isn’t actively used. A process’s thread alternates between actively using its user stack and its kernel stack. The kernel stack is separate (and protected from user code) so that the kernel can execute even if a process has wrecked its user stack.
-->

每个进程都有一个执行线程（或简称线程），它执行进程的指令。
线程可以暂停并稍后恢复。
为了在进程之间透明地切换，内核暂停当前运行的线程并恢复另一个进程的线程。
线程的大部分状态（局部变量，函数调用返回地址）都存储在线程的堆栈中。
每个进程都有两个堆栈：用户堆栈和内核堆栈（`p-> kstack`）。
当进程正在执行用户指令时，只有其用户堆栈正在使用，并且其内核堆栈为空。
当进程进入内核（由于系统调用或中断）时，内核代码在进程的内核堆栈上执行;
当进程在内核中时，其用户堆栈仍包含已保存的数据，但未被主动使用。
进程的线程在主动使用其用户堆栈及其内核堆栈之间交替。
内核堆栈是独立的（并且不受用户代码保护），因此即使进程已破坏其用户堆栈，内核也可以执行。 

<!--
When a process makes a system call, the processor switches to the kernel stack, raises the hardware privilege level, and starts executing the kernel instructions that implement the system call. When the system call completes, the kernel returns to user space: the hardware lowers its privilege level, switches back to the user stack, and resumes executing user instructions just after the system call instruction. A process’s thread can ‘‘block’’ in the kernel to wait for I/O, and resume where it left off when the I/O has finished.
-->

当进程进行系统调用时，处理器切换到内核堆栈，
提高硬件权限级别，并开始执行实现系统调用的内核指令。 
当系统调用完成时，内核返回用户空间：
硬件降低其权限级别，切换回用户堆栈，并在系统调用指令之后继续执行用户指令。
进程的线程可以在内核中“阻塞”以等待 I/O，并在 I/O 完成时从中断处继续。

Figure 1-3. Layout of a virtual address space


<!--
p->state indicates whether the process is allocated, ready to run, running, waiting for I/O, or exiting.
-->

`p->state` 指示进程状态是刚被分配，准备运行，运行，等待 I/O 还是退出中。

<!--
p->pgdir holds the process’s page table, in the format that the x86 hardware expects. xv6 causes the paging hardware to use a process’s p->pgdir when executing that process. A process’s page table also serves as the record of the addresses of the physical pages allocated to store the process’s memory.
-->

`p->pgdir` 以x86硬件期望的格式保存进程的页表。
xv6 导致分页硬件在执行该进程时使用进程的 `p->pgdir`。 
进程的页表还用作分配用于存储进程内存的物理页的地址记录。


## Code: the first address space

<!--
To make the xv6 organization more concrete, we’ll look how the kernel creates the ﬁrst address space (for itself), how the kernel creates and starts the ﬁrst process, and how that process performs the ﬁrst system call. By tracing these operations we see in detail how xv6 provides strong isolation for processes. The ﬁrst step in providing strong isolation is setting up the kernel to run in its own address space.
-->

为了使xv6组织更具体，我们将看看内核如何创建第一个地址空间（为自己），
内核如何创建和启动第一个进程，以及该进程如何执行第一个系统调用。
通过跟踪这些操作，我们详细了解xv6如何为进程提供强大的隔离。
提供强隔离的第一步是将内核设置为在自己的地址空间中运行。

<!--
When a PC powers on, it initializes itself and then loads a boot loader from disk into memory and executes it. Appendix B explains the details. Xv6’s boot loader loads the xv6 kernel from disk and executes it starting at entry (1044). The x86 paging hardware is not enabled when the kernel starts; virtual addresses map directly to physical addresses.
-->

当PC启动时，它会自行初始化，然后将启动加载程序从磁盘加载到内存中并执行它。
附录B解释了细节。
Xv6的引导加载程序从磁盘加载xv6内核并从 `entry`（1044）开始执行它。
内核启动时，x86分页硬件未启用; 虚拟地址直接映射到物理地址。

<!--
The boot loader loads the xv6 kernel into memory at physical address 0x100000. The reason it doesn’t load the kernel at 0x80100000, where the kernel expects to ﬁnd its instructions and data, is that there may not be any physical memory at such a high address on a small machine. The reason it places the kernel at 0x100000 rather than 0x0 is because the address range 0xa0000:0x100000 contains I/O devices.
-->

引导加载程序将 xv6 内核加载到物理地址 `0x100000` 的内存中。
它没有在内核期望找到其指令和数据的 `0x80100000` 加载内核的原因
是在小型机器上的这么高的地址上可能没有任何物理内存。
它将内核置于 `0x100000` 而不是 `0x0` 的原因是因为地址范围 `0xa0000:0x100000`
包含 I/O 设备。

<!--
To allow the rest of the kernel to run, entry sets up a page table that maps virtual
addresses starting at 0x80000000 (called KERNBASE (0207)) to physical addresses starting at 0x0 (see Figure 1-2). Setting up two ranges of virtual addresses that map to the same physical memory range is a common use of page tables, and we will see more  examples like this one.
-->

为了允许内核的其余部分运行，条目设置一个页表，
将从 `0x80000000` 开始的虚拟地址（称为`KERNBASE`（0207））映射到从 `0x0`
开始的物理地址（参见图1-2）。
设置映射到相同物理内存范围的两个虚拟地址范围是页表的常见用法，我们将看到更多这样的示例。

<!--
The entry page table is defined in main.c (1306). We look at the details of page tables in Chapter 2, but the short story is that entry 0 maps virtual addresses main+code 0:0x400000 to physical addresses 0:0x400000. This mapping is required as long as entry is executing at low addresses, but will eventually be removed.
-->

入口页表在 `main.c` 中定义（1306）。
我们在第2章中查看页表的详细信息，但简短的说明是条目0将虚拟地址 
`0：0x400000` 映射到物理地址 `0：0x400000` 。
只要 `entry` 在低地址执行则需要此映射，该映射最终将被删除。

<!--
Entry 512 maps virtual addresses KERNBASE:KERNBASE+0x400000 to physical addresses 0:0x400000. This entry will be used by the kernel after entry has ﬁnished; it maps the high virtual addresses at which the kernel expects to ﬁnd its instructions and data to the low physical addresses where the boot loader loaded them. This mapping restricts the kernel instructions and data to 4 Mbytes.
-->

条目 512 将虚拟地址 `ERNBASE:KERNBASE+0x400000` 映射到物理地址 `0:0x400000`。
输入完成后，内核将使用此条目;
它将内核期望发现其指令和数据的高虚拟地址映射到引导加载程序加载它们的低物理地址。
此映射将内核指令和数据限制为4 MB。

<!--
Returning to entry, it loads the physical address of entrypgdir into control register %cr3. The value in %cr3 must be a physical address. It wouldn’t make sense for %cr3 to hold the virtual address of entrypgdir, because the paging hardware doesn’t know how to translate virtual addresses yet; it doesn’t have a page table yet. The symbol entrypgdir refers to an address in high memory, and the macro V2P_WO (0213) subtracts KERNBASE in order to find the physical address. To enable the paging hardware, xv6 sets the flag CR0_PG in the control register %cr0.
-->

返回 `entry`，它将 `entrypgdir` 的物理地址加载到控制寄存器`%cr3`中。
`%cr3` 中的值必须是物理地址。 
`%cr3` 保留 `entrypgdir` 的虚拟地址是没有意义的，因为分页硬件还不知道如何翻译虚拟地址;
它还没有页表。
符号 `entrypgdir` 指向高内存中的地址，宏 `V2P_WO`（0213）减去 `KERNBASE` 以便找到物理地址。
要启用分页硬件， xv6 会在控制寄存器 `%cr0` 中设置标志 `CR0_PG`。 

<!--
The processor is still executing instructions at low addresses after paging is enabled,
which works since entrypgdir maps low addresses. If xv6 had omitted entry 0 from
entrypgdir, the computer would have crashed when trying to execute the instruction after
the one that enabled paging.
-->

在启用分页后，处理器仍然在低地址执行指令，这是因为 `entrypgdir` 映射了低地址。
如果 xv6 从 `entrypgdir` 中省略了条目 0，则在尝试执行启用分页的指令后，计算机将崩溃。

<!--
Now entry needs to transfer to the kernel’s C code, and run it in high memory. First it
makes the stack pointer, %esp, point to memory to be used as a stack (1058). All symbols
have high addresses, including stack, so the stack will still be valid even when the low
mappings are removed. Finally entry jumps to main, which is also a high address. The
indirect jump is needed because the assembler would otherwise generate a PC-relative
direct jump, which would execute the low-memory version of main. Main cannot return, since
the there’s no return PC on the stack. Now the kernel is running in high addresses in the
function main (1217).
-->

现在，入口需要转移到内核的C代码，并在高内存中运行。
首先，它使堆栈指针 `%esp` 指向要用作堆栈的内存（1058）。
所有符号都是高地址，包括堆栈，因此即使删除了低映射，堆栈仍然有效。
最后输入跳转到 `main`，这也是一个高地址。
需要间接跳转，因为汇编器否则会生成PC相对直接跳转，这将执行 `main` 的低内存版本。
`main` 无法返回，因为堆栈上没有返回PC。
现在内核在函数 `main` 中运行在高地址中（1217）。

## Code: creating the first process

<!--
Now we’ll look at how the kernel creates user-level processes and ensures that they are
strongly isolated.
-->

现在我们将看看内核如何创建用户级进程并确保它们是高度隔离的。

Figure 1-4. A new kernel stack


<!--
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
-->

在 `main`（1217）初始化几个设备和子系统之后，
它通过调用`userinit`（2520）创建第一个进程。
`userinit` 的第一个动作是调用 `allocproc`。
`allocproc`（2473）的工作是在进程表中分配一个槽（`struct proc`）
并初始化其内核线程执行所需的进程状态部分。
为每个新进程调用`Allocproc`，而仅为第一个进程调用 `userinit`。
`Allocproc` 扫描 `proc` 表以查找状态为 `UNUSED`（2480-2482）的插槽。
当它找到一个未使用的槽时，`allocproc` 将状态设置为 `EMBRYO` 以将其标记为已使用，
并为进程提供唯一的 `pid`（2469-2489）。
接下来，它尝试为进程的内核线程分配内核堆栈。
如果内存分配失败，`allocproc` 改变状态恢复到未使用，并返回零表示故障。

<!--
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
-->

现在 `allocproc` 必须设置新进程的内核堆栈。
`allocproc` 特意编写使得在 `fork` 和创建第一个进程时都可以被使用。
`allocproc` 使用专门准备的内核堆栈和一组内核寄存器来设置新进程，
这使得它在首次运行时“返回”用户空间。
准备好的内核堆栈的布局如图1-4所示。
`allocproc` 通过设置返回程序计数器值来完成这项工作的一部分，
这将导致新进程的内核线程首先在 `forkret` 中执行，
然后在 `trapret` 中执行（2507-2512）。
内核线程将从 `p->context` 复制的寄存器内容开始执行。 
因此，将 `p->context->eip` 设置为 `forkret` 将导致内核线程在 `forkret` 的开头执行（2853）。


<!--
As we will see in Chapter 3, the way that control transfers from user software to the kernel is via an interrupt mechanism, which is used by system calls, interrupts, and exceptions. Whenever control transfers into the kernel while a process is running, the hardware and xv6 trap entry code save user registers on the process’s kernel stack. userinit writes values at the top of the new stack that look just like those that would be there if the process had entered the kernel via an interrupt (2533-2539), so that the ordinary code for returning from the kernel back to the process’s user code will work. These values are a struct trapframe which stores the user registers. Now the new process’s kernel stack is completely prepared as shown in Figure 1-4.
-->

正如我们将在第3章中看到的那样，控制从用户软件转移到内核的方式是通过中断机制，
这也是系统调用，中断和异常使用的方法。
每当控制在进程运行时转移到内核中，
硬件和 xv6 陷阱条目代码就会在进程的内核堆栈上保存用户寄存器。
`userinit` 将值写在新堆栈的顶部，看起来就像进程通过中断（2533-2539）进入内核时那样，
以便从内核返回到进程用户的普通代码代码将工作。
这些值是存储用户寄存器的`struct trapframe`。
现在，新进程的内核堆栈已完全准备好，如图1-4所示。 

<!--
The first process is going to execute a small program (`initcode.S`; (8400)). The
process needs physical memory in which to store this program, the program needs to
be copied to that memory, and the process needs a page table that maps user-space
address to that memory.
-->

第一个过程是执行一个小程序（`initcode.S`;（8400））。
该进程需要物理内存来存储该程序，
程序需要复制到该内存，并且该进程需要一个将用户空间地址映射到该内存的页表。

<!--
`userinit` calls `setupkvm`(1818) to create a page table for the process with (at first)
mappings only for memory that the kernel uses. We will study this function in detail
in Chapter 2, but at a high level `setupkvm` and `userinit` create an address space as
shown in Figure 1-2.
-->

`userinit`调用`setupkvm`（1818）为进程创建一个页表，
其中（首先）映射仅用于内核使用的内存。
我们将在第2章详细研究这个函数，
但是大体上 `setupkvm`和`userinit`创建一个地址空间，如图1-2所示。

<!--
The inital contents of the first process's user-space memory are the compiled
form of `initcode.S`; as part of the kernel build process, the linker embeds that binary
in the kernel and defines two special symbols, `_binary_initcode_start` and
`_binary_initcode_size`, indicating the location and size of the binary. `Userinit` copies
that binary into the new process's memory by calling `inituvm`, which allocates one
page of physical memory, maps virtual address zero to that memory, and copies the
binary to that page(1886).
-->

第一个进程的用户空间内存的初始内容是 `initcode.S` 的编译形式; 
作为内核构建过程的一部分，
链接器将该二进制文件嵌入内核并定义两个特殊符号，
`_binary_initcode_start`和`_binary_initcode_size`， 指示二进制文件的位置和大小。
`Userinit` 通过调用 `inituvm` 将二进制文件复制到新进程的内存中，
`inituvm`分配一页物理内存，将虚拟地址零映射到该内存，
并将二进制文件复制到该页面（1886）。

<!--
Then `userinit` sets up the trap frame (0602) with the initial user mode state:
the `%cs` register contains a segment selector for the `SEG_UCODE` segment runing at
privilege level `DPL_USER` (i.e., user mode rather than kernel mode), and similarly
`%ds`, `%es`, and `%ss` use `SEG_UDATA` with privilege `DPL_USER`. The `%eflags` `FL_IF`
is set to allow hardware interrupts; we will reexamine this in Chapter 3.
-->

然后 `userinit` 设置具有初始用户模式状态的陷阱帧（0602）：
`%cs`寄存器包含一个段选择器，在特权级别 “DPL_USER” 运行的`SEG_UCODE`段
（即用户模式而不是内核模式），
类似地，`%ds`，`%es`和`%ss` 使用具有特权级别 `DPL_USER` 的 `SEG_UDATA`。
`%eflags` 的 `FL_IF`设置为允许硬件中断; 我们将在第3章重新审视这一点。

<!--
The stack pointer %esp is set to the process’s largest valid virtual address, p->sz. The instruction pointer is set to the entry point for the initcode, address 0.
-->

堆栈指针 `%esp` 设置为进程的最大有效虚拟地址 `p->sz`。
指令指针设置为 `initcode` 的入口点，地址 0。

<!--
The function userinit sets p->name to initcode mainly for debugging. Setting p->cwd sets the process’s current working directory; we will examine namei in detail in Chapter 6.
-->

函数 `userinit` 将 `p->name` 设置为 `initcode` 主要用于调试。
设置 `p->cwd` 设置进程的当前工作目录; 我们将在第6章详细介绍 `namei`。

<!--
Once the process is initialized, userinit marks it available for scheduling by setting p->state to RUNNABLE.
-->

初始化进程后，`userinit` 通过将 `p->state` 设置为 `RUNNABLE` 来标记它可用于调度。

## Code: Runing the first process



<!--
Now that the ﬁrst process’s state is prepared, it is time to run it. After main calls userinit, mpmain calls scheduler to start running processes (1257). Scheduler (2758) looks for a process with p->state set to RUNNABLE, and there’s only one: initproc. It sets the per-cpu variable proc to the process it found and calls switchuvm to tell the hardware to start using the target process’s page table (1879). Changing page tables while executing in the kernel works because setupkvm causes all processes’ page tables to have identical mappings for kernel code and data. switchuvm also sets up a task state segment SEG_TSS that instructs the hardware to execute system calls and interrupts on the process’s kernel stack. We will re-examine the task state segment in Chapter 3.
-->

现在已经准备好了第一个进程的状态，现在是运行它的时候了。
在 `main` 调用 `userinit`之后，`mpmain` 调用调度程序以开始运行进程（1257）。
调度程序（2758）查找 `p->state` 设置为 `RUNNABLE` 的进程， 并且只有一个：`initproc`。 
它将 per-cpu 变量 `proc` 设置为它找到的进程，
并调用 `switchuvm` 告诉硬件开始使用目标进程的页表（1879）。
在内核中执行时更改页表是有效的，因为 `setupkvm`
会导致所有进程的页表具有相同的内核代码和数据映射。
`switchuvm` 还设置一个任务状态段 `SEG_TSS`，
指示硬件在进程的内核堆栈上执行系统调用和中断。 我们将在第4章重新检查任务状态段。


<!--
scheduler now sets p->state to RUNNING and calls swtch (3059) to perform a
context switch to the target process’s kernel thread. swtch ﬁrst saves the current
registers.
The current context is not a process but rather a special per-cpu scheduler context,
so scheduler tells swtch to save the current hardware registers in per-cpu storage
(cpu->scheduler) rather than in any process’s kernel thread context. swtch then
loads the saved registers of the target kernel thread (p->context) into the x86 hardware
registers, including the stack pointer and instruction pointer. We’ll examine
swtch in more detail in Chapter 5. The ﬁnal ret instruction (3078) pops the target
process’s %eip from the stack, ﬁnishing the context switch. Now the processor is running
on the kernel stack of process p.
-->


`scheduler` 现在将 `p->state` 设置为 `RUNNING` 
并调用 `swtch`（3059）以执行到目标进程的内核线程的上下文切换。
`swtch` 首先保存当前寄存器。 
当前上下文不是一个进程，
而是一个特殊的 per-cpu 调度程序上下文，
因此调度程序告诉 `swtch` 将当前硬件寄存器保存在
per-cpu 存储（`cpu->scheduler`）中，而不是在任何进程的内核线程上下文中。
然后，`swtch` 将目标内核线程（`p->context`）
的已保存寄存器加载到 x86 硬件寄存器中，包括堆栈指针和指令指针。
我们将在第5章中更详细地研究 `swtch`。
最终 ret 指令（3078）从堆栈中弹出目标进程的 `%eip`，完成上下文切换。
现在处理器正在进程 `p` 的内核堆栈上运行。 

<!--
`Allocproc` has previously set initproc's p->context->eip to forkret,
so the ret starts executing forkret.
On the first invocation (that is this one), forkret (2853) runs
initilization functions that cannot be run from main because they
must be run in the context of a regular process with its own kernel stack.
Then forkret returns. Allocproc arranged that the top word on the stack after
p->context is popped off would be trapret, so now trapret begins executing,
with %esp set to p->tf. Trapret (3324) uses pop instructions to restore
registers from the trap frame (0602) just as swtch did with the kernel context:
popal restores the general registers, then the popl instructions restore 
%gs, %fs, %es, and %ds. The addl skips over the two fields 
trapno and errcode. Finally, the iret instruction pops %cs, %eip, %flags, %esp, and
%ss from the stack. The contents of the trap frame have been transferred to the CPU
state, so the processor continues at the %eip speciﬁed in the trap frame. For initproc, that means virtual address zero, the ﬁrst instruction of initcode.S.
-->

`Allocproc` 先前已将 `initproc` 的 `p->context->eip` 设置为 `forkret`，
所以 `ret` 开始执行 `forkret`。
在第一次调用（即这一次）时，`forkret`（2853）运行无法从
`main` 运行的启动函数，因为它们必须在具有自己的内核堆栈的常规进程的上下文中运行。
然后 `forkret` 返回。
`Allocproc` 安排在弹出 `p->context` 之后堆栈的顶部 word
将是 `trapret`，所以现在 `trapret` 开始执行，
`%esp` 设置为 `p->tf`。
`Trapret`（3324）使用 `pop` 指令从陷阱帧（0602）恢复寄存器，就像内核上下文的 `swtch` 一样：
`popal` 恢复通用寄存器，然后 `popl` 指令恢复 `%gs`，`%fs`，`%es`和`%ds`。
`addl` 跳过 `trapno` 和 `errcode` 这两个字段。
最后，`iret`  指令从堆栈中弹出 `%cs`，`%eip`，`%flags`，`%esp` 和 `%ss`。
陷阱帧的内容已转移到CPU状态，因此处理器从陷阱帧中指定的`%eip`继续。
对于 `initproc`，这意味着虚拟地址为零，即`initcode.S`的第一条指令。

<!--
At this point, %eip holds zero and %esp holds 4096. These are virtual addresses in the process’s address space. The processor’s paging hardware translates them into physical addresses. allocuvm has set up the process’s page table so that virtual address zero refers to the physical memory allocated for this process, and set a flag (PTE_U) that tells the paging hardware to allow user code to access that memory. The fact that userinit (2533) set up the low bits of %cs to run the process’s user code at CPL=3 means that the user code can only use pages with PTE_U set, and cannot modify sensitive hardware registers such as %cr3. So the process is constrained to using only its own memory.
-->

此时，`%eip` 保持为零，`%esp` 保持4096.
这些是进程地址空间中的虚拟地址。
处理器的分页硬件将它们转换为物理地址。
 `allocuvm` 有设置进程的页表，以便虚拟地址零指的是为此进程分配的物理内存，
并设置一个标志（`PTE_U`），告诉分页硬件允许用户代码访问该内存。
`userinit`（2533）设置`%cs` 的低位以在 `CPL = 3`处运行进程的用户代码这一事实
意味着用户代码只能使用设置了`PTE_U`的页面，
并且无法修改敏感的硬件寄存器，例如`%cr3`。
因此，该过程仅限于使用自己的内存。


## The first system call: exec

<!--
Now that we have seen how the kernel provides strong isolation for processes, let’s look
at how a user-level process re-enters the kernel to ask for services that it cannot
perform itself.
-->

现在我们已经看到内核如何为进程提供强大的隔离，
让我们看一下用户级进程如何重新进入内核以询问它自身无法执行的服务。

<!--
The first action of initcode.S is to invoke the exec system call. As we saw in Chapter 0,
exec replaces the memory and registers of the current process with a new program, but it
leaves the file descriptors, process id, and parent process unchanged.
-->

`initcode.S` 的第一个动作是调用 `exec` 系统调用。
正如我们在第0章中看到的，`exec` 用新程序替换当前进程的内存和寄存器，
但它保持文件描述符，进程ID和父进程不变。

<!--
Initcode.S (8409) begins by pushing three values on the stack—$argv, $init, and $0—and
then sets %eax to SYS_exec and executes int T_SYSCALL: it is asking the kernel to run the
exec system call. If all goes well, exec never returns: it starts running the program
named by $init, which is a pointer to the NUL-terminated string /init (8422-8424). The
other argument is the argv array of command-line arguments;
the zero at the end of the array marks its end. If the exec fails and does return,
initcode loops calling the exit system call, which deﬁnitely should not return
(8416-8420).
-->

`Initcode.S`（8409）首先在堆栈上 push 三个值 --- `$argv`，`$init` 和 `$0` ---
然后将 `%eax` 设置为 `SYS_exec` 并执行 `int T_SYSCALL` ：它要求内核运行 `exec` 系统调用。
如果一切顺利，`exec` 永远不会返回：它开始运行由 `$init` 命名的程序，
这是一个指向 NUL终止的字符串 `/init`（8422-8424）的指针。 
另一个参数是命令行参数的 `argv` 数组;
数组末尾的零标志着它的结束。
如果 `exec` 失败并且确实返回，
则 `initcode` 循环调用 `exit` 系统调用，该调用无法返回（8416-8420）。

<!--
This code manually crafts the ﬁrst system call to look like an ordinary system call, which
we will see in Chapter 3. As before, this setup avoids special-casing the ﬁrst process (in
this case, its ﬁrst system call), and instead reuses code that xv6 must provide for
standard operation.
-->

这段代码手动地将第一个系统调用看作普通的系统调用，
我们将在第3章中看到。
如前所述，这种设置避免了将第一个进程
（在这种情况下，它的第一个系统调用）作为特殊情况处理，
而是重用 xv6 必须为标准操作提供的代码。

<!--
Chapter 2 will cover the implementation of exec in detail, but at a high level it replaces
initcode with the /init binary, loaded out of the ﬁle system. Now initcode (8400) is done,
and the process will run /init instead. Init (8510) creates a new console device file if
needed and then opens it as ﬁle descriptors 0, 1, and 2. Then it loops, starting a console
shell, handles orphaned zombies until the shell exits, and repeats. The system is up.
-->

第2章将详细介绍 `exec` 的实现，但在较高的层次上，它将使用 `/init` 
二进制文件替换 `initcode` ，从文件系统中加载。
现在 `initcode`（8400）完成了，进程将运行 `/init`。
如果需要，`Init`（8510）创建一个新的控制台设备文件，
然后将其作为文件描述符 0,1和2打开。
然后循环，启动控制台shell，处理孤立的僵尸进程，直到shell退出，并重复。
系统启动了。

## Real world

<!--
In the real world, one can ﬁnd both monolithic kernels and microkernels. Many Unix kernels
are monolithic. For example, Linux has a monolithic kernel, although some OS functions run
as user-level servers (e.g., the windowing system). Kernels such as L4, Minix, QNX are
organized as a microkernel with servers, and have seen wide deployment in embedded
settings.
-->

在现实世界中，人们可以找到宏内核和微内核。
许多Unix内核都是宏内核的。
例如，Linux 使用宏内核，尽管一些 OS 功能作为用户级服务（例如，窗口系统）运行。
诸如 L4，Minix，QNX 之类的内核被组织为具有服务的微内核，并且已经在嵌入式设置中进行了广泛部署。

<!--
Most operating systems have adopted the process concept, and most processes look similar
to xv6’s. A real operating system would ﬁnd free proc structures with an explicit free
list in constant time instead of the linear-time search in allocproc; xv6 uses the linear
scan (the ﬁrst of many) for simplicity.
-->

大多数操作系统都采用了进程的概念，大多数进程看起来与xv6类似。
一个真正的操作系统会使用显式空闲列表在固定时间内找到空闲的proc结构，而不是allocproc中的线性时间搜索;
为简单起见，xv6使用线性扫描（许多中的第一个）。

## Exercises

<!--
1. Set a breakpoint at swtch. Single step with gdb’s stepi through the ret to forkret,
   then use gdb’s finish to proceed to trapret, then stepi until you get to initcode at
   virtual address zero.

2. KERNBASE limits the amount of memory a single process can use, which might be
   irritating on a machine with a full 4 GB of RAM. Would raising KERNBASE allow a process
   to use more memory?
-->

1. 在 `swtch` 设置断点。 单步用 `gdb` 的 `stepi` 通过 `ret` 到 `forkret` ，
    然后使用gdb的 `finish` 继续执行到 `trapret`，然后 `stepi` 直到你到达虚拟地址为零的initcode。

2. `KERNBASE` 限制单个进程可以使用的内存量，这可能会对具有完整4 GB RAM的计算机造成不便。
   提高 `KERNBASE` 会让进程使用更多内存吗？


