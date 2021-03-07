# Part 1: Physical Page Management

There are 5 functions in total to be implemented:
```c
boot_alloc()
mem_init() // only up to the call to check_page_free_list(1)
page_init()
page_alloc()
page_free()
```
So here we go :)

## 1. boot_alloc()

Things seem to be easy by just typing the 3-liner:
```c
result = nextfree;
// increase the virtual address of next byte of free memory by n, and round up to the nearest multiple of PGSIZE
nextfree = ROUNDUP(nextfree + n, PGSIZE);
return result;
```
But note that we also need to handle some **corner cases** :
- If ``n < 0``, ``boot_alloc`` should panic, because you cannot allocate nagative numbers of pages.
- If ``n > 0``, allocates enough pages of contiguous physical memory to hold 'n' bytes. Doesn't initialize the memory. Returns a kernel virtual address.
- If ``n == 0``, returns the address of the next free page without allocating anything.
- If we're **out of memory**, ``boot_alloc`` should panic.

How do we know we're out of memory? Here we have to see ``inc/memlayout.h``, as the header file has already provided the virtual memory map:

<img src="./virtu-mem-map.png" width=500>

Note the part from ``KERNBASE, KSTACKTOP`` to ``4 Gig``, as the space is for JOS to remap the physical memory. 

Given the definition of ``PTSIZE`` in ``inc/mmu.h``:
```c
#define PTSIZE		(PGSIZE*NPTENTRIES) // bytes mapped by a page directory entry
```
And also mind the type definition in ``inc/types.h``:
```c
// Pointers and addresses are 32 bits long.
// We use pointer types to represent virtual addresses,
// uintptr_t to represent the numerical values of virtual addresses,
// and physaddr_t to represent physical addresses.
typedef int32_t intptr_t;
typedef uint32_t uintptr_t;
typedef uint32_t physaddr_t;
```
You may have noticed the macros use the ``uintptr_t`` data type to represent the linear address. So the code to detect whether we're out of memory is:
```c
// one solution
if ((uintptr_t) nextfree >= KERNBASE + PTSIZE) {
    panic("boot_alloc(): out of memory\n");
}
// yet another solution
if ((uintptr_t) nextfree >= KSTACKTOP + PTSIZE) {
    panic("boot_alloc(): out of memory\n");
}
```

Hence, the code that handles the corner cases as a whole is:
```c
static void *
boot_alloc(uint32_t n)
{
	static char *nextfree;	// virtual address of next byte of free memory
	char *result;

	if (!nextfree) {
		extern char end[];
		nextfree = ROUNDUP((char *) end, PGSIZE);
	}

	if (n < 0) {
		panic("boot_alloc(): the pages to be allocated cannot be negative\n");
	}
	result = nextfree;
	if (n == 0) {
		return result;
	}
	nextfree = ROUNDUP(nextfree + n, PGSIZE); 
	if ((uintptr_t) nextfree >= KSTACKTOP + PTSIZE) {
		panic("boot_alloc(): out of memory\n");
	}

	return result;
}
```

## 2. Part of mem_init()
Just code as the instruction mentioned. ``boot_alloc`` is of great help to handle this, and be aware of the size of ``struct PageInfo``.
```c
pages = (struct PageInfo *) boot_alloc(npages * sizeof(struct PageInfo));
memset(pages, 0, npages);
```

## 3. page_init()
Note these variables declared in ``kern/pmap.c`` at the very start, as we will use most of them to implement this function.
```c
// These variables are set by i386_detect_memory()
size_t npages; // Amount of physical memory (in pages)
static size_t npages_basemem; // Amount of base memory (in pages)

// These variables are set in mem_init()
pde_t *kern_pgdir;	// Kernel's initial page directory
struct PageInfo *pages;	// Physical page state array
static struct PageInfo *page_free_list;	// Free list of physical pages
```
Now we need to do four things:
- Mark physical page 0 as in use.
- The rest of base memory, ``[PGSIZE, npages_basemem * PGSIZE)`` is free.
- Then comes the IO hole ``[IOPHYSMEM, EXTPHYSMEM)``, which must never be allocated.
- Then extended memory ``[EXTPHYSMEM, ...)``. Some of it is in use, some is free. Where is the kernel in physical memory?  Which pages are already in use for page tables and other data structures?

Note the data type of ``pages``: ``struct PageInfo``. Its definition is in ``inc/memlayout.h``:
```c
struct PageInfo {
	// Next page on the free list.
	struct PageInfo *pp_link;

	// pp_ref is the count of pointers (usually in page table entries)
	// to this page, for pages allocated using page_alloc.
	// Pages allocated at boot time using pmap.c's
	// boot_alloc do not have valid reference count fields.

	uint16_t pp_ref;
};
```
Yes, it is actually a linked list with the count of pointers. As we know its inner view, it's not hard to code.

But we need to be very careful to initialize the extended memory.
Here we need to find the physical address of the kernel(``KERNBASE`` or ``KSTACKTOP``) and the number of pages in use.

Remind when ``n == 0``, ``boot_alloc(n)`` will return the virtual address of the next free page. We will also use ``PADDR(boot_alloc(0))`` to convert this virtual address to a physical address.

Also note the definition of ``PGSIZE`` in ``inc/mmu.h``, because we have to calculate the number of pages in use:
```c
#define PGSIZE		4096		// bytes mapped by a page
```

So here is the code:

```c
void
page_init(void) {
	// Mark physical page 0 as in use
	pages[0].pp_ref = 1;

	// Map [PGSIZE, npages_basemem * PGSIZE)
	size_t i; // npages_basemem is of type size_t
	for (i = 1; i < npages_basemem; ++i) {
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}

	// Map the IO hole [IOPHYSMEM, EXTPHYSMEM)
	for ( ; i < (size_t) (EXTPHYSMEM / PGSIZE); ++i) {
		pages[i].pp_ref = 1;
	}

	// Map extended memory
	// Note: boot_alloc(0) returns a kernel virtual address, here we need to convert it to a physical address
	physaddr_t first_free_physaddr = PADDR(boot_alloc(0));
	// Calculate the number of pages in use
	size_t first_free_page_num = first_free_physaddr / PGSIZE;
	// Mark the memory used by the kernel and pages in use
	for ( ; i < first_free_page_num; ++i) {
		pages[i].pp_ref = 1;
	}
	// Mark other pages
	for ( ; i < npages; ++i) {
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = & pages[i];
	}
}
```

## 4. page_alloc()

We just code as the instruction mentioned again. Note that the function ``page2kva`` defined in ``kern/pmap.h`` returns the corresponding kernel virtual address(which is ``(pp - pages) << PGSHIFT + KERNBASE``), and the data type is a void pointer.
```c
struct PageInfo *
page_alloc(int alloc_flags) {
	struct PageInfo* pp;

	if (!page_free_list) {
		return NULL;
	}
	pp = page_free_list;
	// Make page_free_list to point its next page
	page_free_list = page_free_list->pp_link
	pp->pp_link = NULL;

	if (alloc_flags & ALLOC_ZERO) {
		void * va = page2kva(pp);
		memset(va, '\0', PGSIZE);
	}
	return pp;
}
```

## 5. page_free()

Make sure the page to free points to ``page_free_list``, and be careful about the corner cases.
```c
void
page_free(struct PageInfo *pp) {
	// Fill this function in
	// Hint: You may want to panic if pp->pp_ref is nonzero or
	// pp->pp_link is not NULL.
	if (pp->pp_ref != 0) {
		panic("page_free(): the page is still in use!\n");
	}
	if (pp->pp_link != NULL) {
		panic("page_free(): the page is empty.\n");
	}
	pp->pp_link = page_free_list;
	page_free_list = pp;
}
```

## Finish Up

To test whether the functions work, we have to remove the line ``panic("mem_init: This function is not finished\n");``. Then run ``make qemu`` in the lab folder. If all the functions behave normally, the terminal would goes like this:
```
check_page_free_list() succeeded!
check_page_alloc() succeeded!
kernel panic at kern/pmap.c:745: assertion failed: page_insert(kern_pgdir, pp1, 0x0, PTE_W) < 0
```
Don't worry about the kernel panic, because we have not finished the function ``page_insert`` yet.