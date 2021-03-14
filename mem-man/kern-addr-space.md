# Part 3: Kernel Address Space

## The Other Part of mem_init()

The only thing we need to do is to set up virtual memory. Here we use ``boot_map_region`` to handle the tasks. 

Here we look back at its parameters:
```c
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
```

And also the virtual memory map:

<img src="./virtu-mem-map.png" width=500>

Remind that ``boot_map_region`` has already set the flag ``PTE_P`` implictly.

DOUBT:
> We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
> to be the kernel stack, but break this into two pieces:
> * ``[KSTACKTOP-KSTKSIZE, KSTACKTOP)`` -- backed by physical memory
> * ``[KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE)`` -- not backed; so if the kernel overflows its stack, it will fault rather than overwrite memory.  Known as a "guard page". 
> 
> Permissions: kernel RW, user NONE

It seems that there are **two** blocks to set up.

```c
//////////////////////////////////////////////////////////////////////
	// Map 'pages' read-only by the user at linear address UPAGES
	// Permissions:
	//    - the new image at UPAGES -- kernel R, user R
	//      (ie. perm = PTE_U | PTE_P)
	//    - pages itself -- kernel RW, user NONE
	// Your code goes here:
	boot_map_region(kern_pgdir, UPAGES, PTSIZE, PADDR(pages), PTE_U);

	//////////////////////////////////////////////////////////////////////
	// Use the physical memory that 'bootstack' refers to as the kernel
	// stack.  The kernel stack grows down from virtual address KSTACKTOP.
	// We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
	// to be the kernel stack, but break this into two pieces:
	//     * [KSTACKTOP-KSTKSIZE, KSTACKTOP) -- backed by physical memory
	//     * [KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE) -- not backed; so if
	//       the kernel overflows its stack, it will fault rather than
	//       overwrite memory.  Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	// Your code goes here:
	boot_map_region(kern_pgdir, KSTACKTOP - KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);

	//////////////////////////////////////////////////////////////////////
	// Map all of physical memory at KERNBASE.
	// Ie.  the VA range [KERNBASE, 2^32) should map to
	//      the PA range [0, 2^32 - KERNBASE)
	// We might not have 2^32 - KERNBASE bytes of physical memory, but
	// we just set up the mapping anyway.
	// Permissions: kernel RW, user NONE
	// Your code goes here:
	boot_map_region(kern_pgdir, KERNBASE, 0xffffffff - KERNBASE + 1, 0, PTE_W);
```

Once the mapping succeeds, qemu will show these messages:
```
check_page_free_list() succeeded!
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_free_list() succeeded!
check_page_installed_pgdir() succeeded!
```

## HomeWork

2. To solve this question, we need to look at virtual memory map in ``inc/memlayout.h`` again. 

Remind that we have already set up virtual memory of ``UPAGES``, ``[KSTACKTOP - KSTKSIZE, KSTACKTOP)`` and ``[KERNBASE, 2^32)``. To get the entry, we have to calculate the value of ``VIRTUAL_ADDR / PTSIZE``. Also note that ``PTSIZE`` is ``8 * PGSIZE``, and ``PGSIZE == 4096``.

So here comes the result:

| Entry | Base Virtual Address | Points to (logically): |
|-|-|-|
| 1023 | 0xffbfffff | Page table for top 4MB of phys memory |
| 1022 | 0xff7fffff | Page table for top (4MB, 8MB] of phys memory |
| 960 | 0xf0000000(KSTACKTOP) | Kernel stack top |
| 959 | 0xefff8000(KSTACKTOP - KSTKSIZE) | Kernel stack base|
| 956 | 0xef000000(UPAGES) | Read-only copies of the Page structures |
| 2 | 0x00800000 | Temporary page mappings |
| 1 | 0x00400000 | User STAB data |
| 0 | 0x00000000 | [see next question] |

<br>

3. To load the user program, the kernel has to **allocate the stack for the user** and **free the memory if the program exits**; Also, if there are multiple programs run concurrently, it's the kernel's responsibility to switch among different threads. There is no copy of the kernel's address space, if the user data overwrites the kernel, it will cause great damage to the kernel and cannot be recovered.
<br>
There is physical memory allocated as the kernel stack, and the attribute named ``PTE_U`` in the kernel stack is not set up to protect the kernel from user access.

4. The maximum amount of basic physical memory supported is ``PTSIZE * PTENTRIES``, which is ``1024 * 1024 * 4K == 4G``. But note that there could be a different case depending on available base & extended momory, this function from ``kern/pmap.c`` shows that it could be the case:
	```c
	static void
	i386_detect_memory(void)
	{
		size_t basemem, extmem, ext16mem, totalmem;

		// Use CMOS calls to measure available base & extended memory.
		// (CMOS calls return results in kilobytes.)
		basemem = nvram_read(NVRAM_BASELO);
		extmem = nvram_read(NVRAM_EXTLO);
		ext16mem = nvram_read(NVRAM_EXT16LO) * 64;

		// Calculate the number of physical pages available in both base
		// and extended memory.
		if (ext16mem)
			totalmem = 16 * 1024 + ext16mem;
		else if (extmem)
			totalmem = 1 * 1024 + extmem;
		else
			totalmem = basemem;

		npages = totalmem / (PGSIZE / 1024);
		npages_basemem = basemem / (PGSIZE / 1024);

		cprintf("Physical memory: %uK available, base = %uK, extended = %uK\n",
			totalmem, basemem, totalmem - basemem);
	}
	```

5. Now look at the ``inc/mmu.h`` for paging data structures and constants: there are 1024 page directory entries and 1024 page table entries per page table(since ``#define NPDENTRIES	1024`` and ``#define NPTENTRIES	1024``). To get the result, we have to calculate the total memory of **the page, page table entries** and **page directory**, in other words, ``1024 * PGSIZE + 1024 * sizeof(pde_t *kern_pgdir) + 1024 * 1024 * sizeof(struct PageInfo *pages)``. The result is ``4M + 4K + 8M``(Note that the complier would do padding for the struct, so the size of a ``struct PageInfo *`` is 8B rather than 6B).
   
6. Note the two instructions in ``entry.S``:
	```assembly
	# Now paging is enabled, but we're still running at a low EIP
	# (why is this okay?).  Jump up above KERNBASE before entering
	# C code.
	mov	$relocated, %eax
	jmp	*%eax
	```
	At this point we do transition to running at an ``EIP`` above ``KERNBASE``.
	<br>
	We **mapped the first 4MB of physical memory** at the virtual address ``[KERNBASE, KERNBASE+4MB)`` as the entry, and making it possible for us to continue executing at a low ``EIP`` between when we enable paging and when we begin running at an ``EIP`` above ``KERNBASE``. It is critical for ``entry.S`` to execute the instructions mentioned above.

## Challenges

To be finished