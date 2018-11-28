# Page tables

<!--
Page tables are the mechanism through which the operating system controls what memory
addresses mean. They allow xv6 to multiplex the address spaces of different processes onto
a single physical memory, and to protect the memories of different processes. The level of
indirection provided by page tables allows many neat tricks. xv6 uses page tables
primarily to multiplex address spaces and to protect memory. It also uses a few simple
page-table tricks: mapping the same memory (the kernel) in several address spaces, mapping
the same memory more than once in one address space (each user page is also mapped into
the kernel’s physical view of memory), and guarding a user stack with an unmapped page.
The rest of this chapter explains the page tables that the x86 hardware provides and how
xv6 uses them. Compared to a real-world operating system, xv6’s design is restrictive, but
it does illustrate the key ideas.
-->


页表是操作系统控制内存地址的意义的机制。
它们允许 xv6 将不同进程的地址空间复用到单个物理内存中，并保护不同进程的内存。
页表提供的间接级别允许许多巧妙的技巧。 xv6主要使用页表来复用地址空间并保护内存。
它还使用了一些简单的页表技巧：在几个地址空间中映射相同的内存（内核），
在一个地址空间中多次映射相同的内存（每个用户页面也映射到内核的内存物理视图）
，并使用未映射的页面保护用户堆栈。
本章的其余部分将介绍x86硬件提供的页表以及xv6如何使用它们。
与现实世界的操作系统相比，xv6的设计具有限制性，但它确实说明了关键思想。


## Paging hardware

<!--
As a reminder, x86 instructions (both user and kernel) manipulate virtual addresses. The
machine’s RAM, or physical memory, is indexed with physical addresses. The x86 page table
hardware connects these two kinds of addresses, by mapping each virtual address to a
physical address.
-->

回忆一下，x86指令（用户和内核）操纵虚拟地址。机器的RAM或物理内存使用物理地址编制索引。
x86页表硬件通过将每个虚拟地址映射到物理地址来连接这两种地址。

<!--
An x86 page table is logically an array of 2^20 (1,048,576) page table entries (PTEs).
Each PTE contains a 20-bit physical page number (PPN) and some flags. The paging hardware
translates a virtual address by using its top 20 bits to index into the page table to ﬁnd
a PTE, and replacing the address’s top 20 bits with the PPN in the PTE. The paging
hardware copies the low 12 bits unchanged from the virtual to the translated physical
address. Thus a page table gives the operating system control over virtual-to-physical
address translations at the granularity of aligned chunks of 4096 (2^12) bytes. Such a
chunk is called a page.
-->

x86页表在逻辑上是
2^20（1,048,576）个页表条目（PTE）的数组。
每个PTE包含一个20位物理页码（PPN）和一些文件。
寻呼硬件通过使用其前20位来索引到页表以发现 PTE，
并用 PTE 中的 PPN 替换地址的前 20 位来转换虚拟地址。
分页硬件将虚拟地址翻译到物理地址时低12位直接复制。
因此，页表使操作系统以4096（2^12）
字节的对齐块的粒度控制虚拟到物理地址转换。这样的块称为页面。

<!--
As shown in Figure 2-1, the actual translation happens in two steps. A page table is
stored in physical memory as a two-level tree. The root of the tree is a 4096-byte page
directory that contains 1024 PTE-like references to page table pages. Each page table page
is an array of 1024 32-bit PTEs. The paging hardware uses the top 10 bits of a virtual
address to select a page directory entry. If the page directory entry is present, the
paging hardware uses the next 10 bits of the virtual address to select a PTE from the page
table page that the page directory entry refers to. If either the page directory entry or
the PTE is not present, the paging hardware raises a fault. This two-level structure
allows a page table to omit entire page table pages in the common case in which large
ranges of virtual addresses have no mappings.
-->

如图2-1所示，实际转换分两步进行。
页表作为两级树存储在物理存储器中。
树的根是一个4096字节的页面目录，其中包含对页表页面的1024个类似PTE的引用。
每个页表页面是1024个32位PTE的数组。
分页硬件使用虚拟地址的前10位来选择页面目录条目。
如果存在页面目录条目，
则分页硬件使用虚拟地址的下一个10位从页面目录条目引用的页面表页面中选择 PTE。
如果页面目录条目或 PTE 不存在，则分页硬件引发故障。
这种两级结构允许页表在大范围的虚拟地址没有映射的常见情况下省略整个页表页。


<!--
Each PTE contains flag bits that tell the paging hardware how the associated virtual
address is allowed to be used. PTE_P indicates whether the PTE is present: if it is
not set, a reference to the page causes a fault (i.e. is not allowed). PTE_W controls
whether instructions are allowed to issue writes to the page; if not set, only reads and
instruction fetches are allowed. PTE_U controls whether user programs are allowed to use
the page; if clear, only the kernel is allowed to use the page. Figure 2-1 shows how it
all works. The ﬂags and all other page hardware related structures are deﬁned in mmu.h
(0700).
-->


<!--
A few notes about terms. Physical memory refers to storage cells in DRAM. A byte of
physical memory has an address, called a physical address. Instructions use only virtual
addresses, which the paging hardware translates to physical addresses, and then sends to
the DRAM hardware to read or write storage. At this level of discussion there is no such
thing as virtual memory, only virtual addresses.
-->






