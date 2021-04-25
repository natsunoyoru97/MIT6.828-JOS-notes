# Lab 3, Part B: Page Faults, Breakpoints Exceptions, and System Calls



## Handling Page Faults

### 1. trap_dispatch
First, we need to handle **page faults**. To do this, we have to detect the type of traps. Go to ``inc/trap.h`` to find the attribute:

```c
struct Trapframe {
  	...
		uint32_t tf_trapno;
  	...
}
```

And also in ``kern/trap.c``:

```c
void
print_trapframe(struct Trapframe *tf)
{
	...
	cprintf("  trap 0x%08x %s\n", tf->tf_trapno, trapname(tf->tf_trapno));
  ...
}
```

So we will use ``tf->tf_trapno`` to identify the type of trap. This function will jump to different code blocks depending on ``tf->tf_trapno == T_PGFLT`` or not. Here we use the switch block to be consistent with other codes such as ``cga_putc()`` in ``kern/console.c `` and ``check_kern_pgdir()`` in ``kern/pmap.c``. Here is the code:

```c
static void
trap_dispatch(struct Trapframe *tf)
{
    int32_t ret;
		// Handle processor exceptions.
		// LAB 3: Your code here.
    switch (tf->tf_trapno) {
        case (T_PGFLT):
            page_fault_handler(tf);
            break;
        default:
            // Unexpected trap: The user process or the kernel has a bug.
            print_trapframe(tf);
            if (tf->tf_cs == GD_KT)
                panic("unhandled trap in kernel");
            else {
                env_destroy(curenv);
                return;
            }
    }
}
```



## The Breakpoint Exception

### 2. trap_dispatch

We have also to handle the **breakpoint exception**, and we will abuse this exception slightly by turning into a **primitive pseudo-system call** that any user environment can use to **invoke the JOS kernel monitor** (note that the function to invoke monitor is in ``kern/monitor.c``). This is quite similiar to what we did to set up the page fault:

```c
static void
trap_dispatch(struct Trapframe *tf)
{
    int32_t ret; 
    // Handle processor exceptions.
    // LAB 3: Your code here.
    switch (tf->tf_trapno) {
        ...
        case (T_BRKPT):
            monitor(tf);
            break;
        ...
    }
}
```



### Questions

**3. The break point test case will either generate a break point exception or a general protection fault depending on how you initialized the break point entry in the IDT (i.e., your call to `SETGATE` from `trap_init`).  Why? How do you need to set it up in order to get the breakpoint exception to work as specified above and what incorrect setup would cause it to trigger a general protection fault?**

Because there is the Descriptor Privilege Level (DPL) to execute the instructions. The break point test case will genereate a break point exception when the DPL is 3 (user mode) and generate a general protection fault when the DPL is 0 (kernel mode). 



**4. What do you think is the point of these mechanisms, particularly in light of what the `user/softint` test program does?**

It depends on the DPL you have initialzed in the IDT. To execute a program without general protection fault, **the CPL (Current Privilege Level) must be prior to or equal to DPL.** If the user program tries to execute an instruction in the kernel mode, it will generate a general protection fault.



## System Calls

Since we have already implemented the handler for interrupt vector ``T_SYSCALL`` in ``kern/trapentry.S`` and ``kern/trap.c`` (see [Lab3, Part A](user-envs-a.md)), here we only change ``trap_dispatch()``.

### 3.1 syscall

Note what has been declared in ``inc/syscall.h``:

```c
/* system call numbers */
enum {
	SYS_cputs = 0,
	SYS_cgetc,
	SYS_getenvid,
	SYS_env_destroy,
	NSYSCALLS
};
```

And then we call the function corresponding to the 'syscall' parameter according to these system call numbers. Note that ``sys_cputs`` has parameters, so we need find out the way to handle them, take a look at ``lib/syscall.h``:

```c
static inline int32_t
syscall(int num, int check, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
    int32_t ret;

    // Generic system call: pass system call number in AX,
    // up to five parameters in DX, CX, BX, DI, SI.
    // Interrupt kernel with T_SYSCALL.
    //
    // The "volatile" tells the assembler not to optimize
    // this instruction away just because we don't use the
    // return value.
    //
    // The last clause tells the assembler that this can
    // potentially change the condition codes and arbitrary
    // memory locations.

    asm volatile("int %1\n"
           : "=a" (ret)
           : "i" (T_SYSCALL),
             "a" (num),
             "d" (a1),
             "c" (a2),
             "b" (a3),
             "D" (a4),
             "S" (a5)
           : "cc", "memory");
    ...
}
```

Since the ``ret``, ``T_SYSCALL`` and ``num`` are reserved parameters, the other five parameters will be in consideration. So we pass ``a1`` and ``a2`` as the first two parameters.

Here is the code:

```c
// Dispatches to the correct kernel function, passing the arguments.
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
    // Call the function corresponding to the 'syscallno' parameter.
    // Return any appropriate return value.
    // LAB 3: Your code here.

    //panic("syscall not implemented");

    switch (syscallno) {
        case (SYS_cputs):
            // pass first two parameters
            sys_cputs((const char *) a1, a2);
            return 0;
        case (SYS_cgetc):
          	return sys_cgetc();
        case (SYS_getenvid):
          	return sys_getenvid();
        case (SYS_env_destroy):
          	return sys_env_destroy(a1);
        default:
          	return -E_INVAL;
    }
}
```



### 3.2. trap_dispatch

Note the comment of ``syscall`` function in 3.1. and the explaination in the document:

> The system call number will go in `%eax`, and the arguments (up to five of them) will go in `%edx`, `%ecx`, `%ebx`, `%edi`, and `%esi`, respectively.  The kernel passes the return value back in `%eax`.

Here we have to bind the value of each register to each parameter. Also note that we have to handle  return value of the ``syscall`` function, in other words, pass the return value back in ``%eax``.

```c
static void
trap_dispatch(struct Trapframe *tf)
{
    int32_t ret; 
    // Handle processor exceptions.
    // LAB 3: Your code here.
    switch (tf->tf_trapno) {
        ...
        case (T_SYSCALL):
            ret = syscall(
                tf->tf_regs.reg_eax,
                tf->tf_regs.reg_edx,
                tf->tf_regs.reg_ecx,
                tf->tf_regs.reg_ebx,
                tf->tf_regs.reg_edi,
                tf->tf_regs.reg_esi);
						// Pass the return value back in %eax
            tf->tf_regs.reg_eax = ret;
            break;
        ...
    }
}
```



## User-mode Startup

### 4. libmain

What we have to do is to set ``thisenv`` to point at our ``Env`` structure in ``envs[]``. To fetch the index, there is a macro called ``ENVX(envid)`` to call:

> The environment index ENVX(eid) equals the environment's index in the 'envs[]' array. 

```c
#define ENVX(envid)		((envid) & (NENV - 1))
```

So the code is:

```c
void
libmain(int argc, char **argv) 
{
    // set thisenv to point at our Env structure in envs[].
    // LAB 3: Your code here.
    thisenv = &envs[ENVX(sys_getenvid())];

    // save the name of the program so that panic() can use it
    if (argc > 0)
      	binaryname = argv[0];

    // call user main routine
    umain(argc, argv);

    // exit gracefully
    exit();
}

```



## Page Faults And Memory Protection

### 5.1. page_fault_handler

Here we have to check if the page fault hapens in kernel mode. 

> Hint: to determine whether a fault happened in user mode or in kernel mode, check the low bits of the `tf_cs`. 

Note the ``%cs`` has 4 bits, and we have to check the low 2 bits. Mind the comments in ``kern/env.c``:

> In particular, the last argument to the SEG macro used in the definition of gdt specifies the Descriptor Privilege Level (DPL) of that descriptor: 0 for kernel and 3 for user. 
>
> The low 2 bits of each segment register contains the Requestor Privilege Level (RPL); 3 means user mode. 

```c
void
page_fault_handler(struct Trapframe *tf)
{
    uint32_t fault_va;

    // Read processor's CR2 register to find the faulting address
    fault_va = rcr2();

    // Handle kernel-mode page faults.

    // LAB 3: Your code here.
    if ((tf->tf_cs & 3) == 0)) {
      	panic("page_fault_handler: page is in kernel mode, fault at %d", fault_va);
    } 
    ...
}
```



### 5.2. user_mem_check

Things to do according to the comment above ``user_mem_check``:

- Page-align the start and the end memory address (i.e. ``va`` and ``va + len``);
- Retrieve the pointer to the page table entry (PTE) and check if these pages can access a virtual address, including:
  - The address is below ``ULIM``;
  - The page exists;
  - The page table gives it permission;
- If there is an error, set ``user_mem_check_addr`` to the first errorneous virtual address, and return ``-E_FAULT`` ;
- If the user program can access this range of addresses, returns ``0``.



So the code is:

```c
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
    // LAB 3: Your code here.
    uintptr_t start = (uintptr_t) ROUNDDOWN(va, PGSIZE);
    uintptr_t end = (uintptr_t) ROUNDUP((va + len), PGSIZE);
    pte_t *curr_pg = NULL;

    // Find the permissions in the range of virtual address of the pages
    for ( ; start < end; start += PGSIZE) {
        curr_pg = pgdir_walk(env->env_pgdir, (void *) start, 0);
        if (start > ULIM || curr_pg == NULL ||
            ((*curr_pg) & perm) != perm) {
            user_mem_check_addr = (start == (uintptr_t) ROUNDDOWN(va, PGSIZE)) ?
                         (uintptr_t) va : (uintptr_t) start;
            return -E_FAULT;
        }
    }

    return 0;
}

```



### 5.3. sys_cputs

Just add one line to call ``user_mem_assert`` to implement sanity check:

```c
static void
sys_cputs(const char *s, size_t len)
{
	...
	// LAB 3: Your code here.
	user_mem_assert(curenv, s, len, 0);
	...
}
```



### 5.4. debuginfo_eip

> Finally, change `debuginfo_eip` in `kern/kdebug.c` to call `user_mem_check` on `usd`, `stabs`, and `stabstr`.  If you now run `user/breakpoint`, you should be able to run backtrace from the kernel monitor and see the backtrace traverse into `lib/libmain.c` before the kernel panics with a page fault. 

Here is the code. Mind the ``-E_FAULT`` is ``6`` and ``user_mem_check`` will return 0 when it is successful.

```c
int
debuginfo_eip(uintptr_t addr, struct Eipdebuginfo *info)
{
    ...
    // Find the relevant set of stabs
    if (addr >= ULIM) {
      	...
    } else {
        // The user-application linker script, user/user.ld,
        // puts information about the application's stabs (equivalent
        // to __STAB_BEGIN__, __STAB_END__, __STABSTR_BEGIN__, and
        // __STABSTR_END__) in a structure located at virtual address
        // USTABDATA.
        const struct UserStabData *usd = (const struct UserStabData *) USTABDATA;

        // Make sure this memory is valid.
        // Return -1 if it is not.  Hint: Call user_mem_check.
        // LAB 3: Your code here.
        if (user_mem_check(curenv, usd, sizeof(const struct UserStabData *), PTE_U) != 0) {
          	return -1;
        }

        stabs = usd->stabs;
        stab_end = usd->stab_end;
        stabstr = usd->stabstr;
        stabstr_end = usd->stabstr_end;

        // Make sure the STABS and string table memory is valid.
        // LAB 3: Your code here.
        if (user_mem_check(curenv, stabs, stab_end - stabs, PTE_U) != 0 ||
          user_mem_check(curenv, stabstr, stabstr_end - stabstr, PTE_U) != 0) {
          	return -1;
        }
    }
}
```





This completes the lab. At this time, you will pass all of the ``make grade`` tests.