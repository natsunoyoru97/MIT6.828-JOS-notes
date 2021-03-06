# Check Whether Your Code Works

Mind this code snippet in ``kern/monitor.c``:
```c
static struct Command commands[] = {
	{ "help", "Display this list of commands", mon_help },
	{ "kerninfo", "Display information about the kernel", mon_kerninfo }
};
```
You would find it helpful to test your code.

## A Small Example
