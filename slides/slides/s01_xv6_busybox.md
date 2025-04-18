---
title: xv6, UNIX Bootup, and systemd
---

# xv6, UNIX Bootup, and Management

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2024, All rights reserved

</div>

---

## `xv6` simplicity

- What *choices and coding techniques* enable `xv6` to be so simple?
- What choices enable it to *avoid the complexity of other systems*?

---

How is it so simple, how does it avoid complexity?

> Is this a minimal number of system calls? Why don't all OSes have a minimal number?

> Why is there no `creat` call?

> This does most of what we need! Why are systems more complex?

> Surprising that `dup` returns the lowest available fd #. What is `dup` used for?

> Why spinlocks over mutexes? Why mutexes over spinlocks?

> Why is the stack at addresses lower than the heap?

> Why does `kmalloc` use `5` instead of `0` in initializing its new page?

---

```c [3|12-19|99-100|124-125|37|60-70|84-96]
struct cpu cpus[NCPU];

struct proc proc[NPROC];

/* ... */

static struct proc *
allocproc(void)
{
	struct proc *p;

	for (p = proc; p < &proc[NPROC]; p++) {
		acquire(&p->lock);
		if (p->state == UNUSED) {
			goto found;
		} else {
			release(&p->lock);
		}
	}
	return 0;

found:
	p->pid   = allocpid();
	p->state = USED;

	// Allocate a trapframe page.
	if ((p->trapframe = (struct trapframe *)kalloc()) == 0) {
		freeproc(p);
		release(&p->lock);
		return 0;
	}

    // GWU additions: For now, assume that this process (thread)
    // always uses its own page-table.
    p->pagetable_proc = p;
	// An empty user page table.
	p->pagetable = proc_pagetable(p);
	if (p->pagetable == 0) {
		freeproc(p);
		release(&p->lock);
		return 0;
	}

	// Set up new context to start executing at forkret,
	// which returns to user space.
	memset(&p->context, 0, sizeof(p->context));
	p->context.ra = (uint64)forkret;
	p->context.sp = p->kstack + PGSIZE;

	return p;
}

/* ... */
pagetable_t
proc_pagetable(struct proc *p)
{
	pagetable_t pagetable;

	// An empty page table.
	pagetable = uvmcreate();
	if (pagetable == 0) return 0;

	// map the trampoline code (for system call return)
	// at the highest user virtual address.
	// only the supervisor uses it, on the way
	// to/from user space, so not PTE_U.
	if (mappages(pagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) < 0) {
		uvmfree(pagetable, 0);
		return 0;
	}

	// map the trapframe page just below the trampoline page, for
	// trampoline.S.
	if (mappages(pagetable, TRAPFRAME, PGSIZE, (uint64)(p->trapframe), PTE_R | PTE_W) < 0) {
		uvmunmap(pagetable, TRAMPOLINE, 1, 0);
		uvmfree(pagetable, 0);
		return 0;
	}

	return pagetable;
}

/* ... */
void *
kalloc(void)
{
	struct run *r;

	acquire(&kmem.lock);
	r = kmem.freelist;
	if (r) kmem.freelist = r->next;
	release(&kmem.lock);

	if (r) memset((char *)r, 5, PGSIZE); // fill with junk
	return (void *)r;
}

/* ... */
int
fork(void)
{
	int          i, pid;
	struct proc *np;
	struct proc *p = myproc();

	// Allocate process.
	if ((np = allocproc()) == 0) { return -1; }

	// Copy user memory from parent to child.
	if (uvmcopy(p->pagetable, np->pagetable, p->sz) < 0) {
		freeproc(np);
		release(&np->lock);
		return -1;
	}
	np->sz = p->sz;

	// copy saved user registers.
	*(np->trapframe) = *(p->trapframe);

	// Cause fork to return 0 in the child.
	np->trapframe->a0 = 0;

	// increment reference counts on open file descriptors.
	for (i = 0; i < NOFILE; i++)
		if (p->ofile[i]) np->ofile[i] = filedup(p->ofile[i]);
	np->cwd = idup(p->cwd);

	safestrcpy(np->name, p->name, sizeof(p->name));

	pid = np->pid;

	release(&np->lock);

	acquire(&wait_lock);
	np->parent = p;
	release(&wait_lock);

	acquire(&np->lock);
	np->state = RUNNABLE;
	release(&np->lock);

	return pid;
}
```

Notes:
1. Static allocation of core structures
2. Simple "allocation" based on states
3. Inline structures within those (static size)
4. Where dynamic allocation is necessary, use only page-sized allocations
5. And use a single simple freelist

---

<div class="multicolumn"><div>

## Simple set of orthogonal syscalls

</div><div>

```c []
// System call numbers
#define SYS_fork 1
#define SYS_exit 2
#define SYS_wait 3
#define SYS_pipe 4
#define SYS_read 5
#define SYS_kill 6
#define SYS_exec 7
#define SYS_fstat 8
#define SYS_chdir 9
#define SYS_dup 10
#define SYS_getpid 11
#define SYS_sbrk 12
#define SYS_sleep 13
#define SYS_uptime 14
#define SYS_open 15
#define SYS_write 16
#define SYS_mknod 17
#define SYS_unlink 18
#define SYS_link 19
#define SYS_mkdir 20
#define SYS_close 21
```

</div></div>

---

# Pivot: System Boot + Management

-v-

- https://en.wikipedia.org/wiki/Systemd
- https://opensource.com/article/20/4/systemd
- https://ewontfix.com/14/

---

## How does a system boot?

1. HW + firmware (POST)
2. Kernel initialization
3. Create process context for `pid 1` or `/sbin/init` :clown_face:
4. Load modules & device drivers (e.g. for disk)
5. Mount the file systems in `/etc/fstab` (`/dev/sd0` $\to$ `/`)
6. Potentially load more modules/device drivers
7. Execute per-service/application initialization (e.g. start daemons)
8. This includes the GUI or a login prompt/ssh server

---

```c [10-33|4-7|37-54|42]
// a user program that calls exec("/init")
// assembled from ../user/initcode.S
// od -t xC ../user/initcode
uchar initcode[] = {0x17, 0x05, 0x00, 0x00, 0x13, 0x05, 0x45, 0x02, 0x97, 0x05, 0x00, 0x00, 0x93,
                    0x85, 0x35, 0x02, 0x93, 0x08, 0x70, 0x00, 0x73, 0x00, 0x00, 0x00, 0x93, 0x08,
                    0x20, 0x00, 0x73, 0x00, 0x00, 0x00, 0xef, 0xf0, 0x9f, 0xff, 0x2f, 0x69, 0x6e,
                    0x69, 0x74, 0x00, 0x00, 0x24, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00};

// Set up first user process.
void
userinit(void)
{
	struct proc *p;

	p        = allocproc();
	initproc = p;

	// allocate one user page and copy initcode's instructions
	// and data into it.
	uvmfirst(p->pagetable, initcode, sizeof(initcode));
	p->sz = PGSIZE;

	// prepare for the very first "return" from kernel to user.
	p->trapframe->epc = 0;      // user program counter
	p->trapframe->sp  = PGSIZE; // user stack pointer

	safestrcpy(p->name, "initcode", sizeof(p->name));
	p->cwd = namei("/");

	p->state = RUNNABLE;

	release(&p->lock);
}

/* from userinit.S... */

# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall

		/*...*/
# char init[] = "/init\0";
init:
  .string "/init\0"

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .quad init
  .quad 0
```

---

## SystemV Bootscripts

```mermaid
%%{init: {"flowchart": {"htmlLabels": false}} }%%
flowchart LR
	A(Hardware)
	B(Kernel)
	C("init (pid 1)")
	C1("Execute commands in /etc/inittab")
	D("rc.sysinit")
	D1("Load kernel modules/drivers (/etc/rc.modules)")
	D2("Mount FSes (in /etc/fstab)")
	E("/etc/rc.d/rc")
	F("Run all /etc/rc.d/rcX.d scripts")

	A --> B
	B --> C
	C --> C1
	C1 --> D
	C --> E
	D --> D1
	D --> D2
	E --> F
```

Outside of [init](https://github.com/mirror/busybox/tree/master/init), these are shell scripts.

---

## SystemV Boot (e.g. Busybox)

Designed around

- a simple C `init`,
- shell scripts to do the rest

Documented in [LFS](https://www.linuxfromscratch.org/lfs/view/9.1-rc1/chapter07/usage.html), and [redhat](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/4/html/reference_guide/s2-boot-init-shutdown-init#s2-boot-init-shutdown-init).

https://en.wikipedia.org/wiki/Booting_process_of_Linux

---

## `inetd`

Start network services lazily, when they are requested

- mapping of port $\to$ service binary
- create sockets for each port awaiting connections
- reboot service if it fails

---

## Middleware

We should have system management software
- to handle boot
- *and* to ease system coordination

---

## `systemd`

A hub for communication and coordination
- domain sockets
- pub/sub
- lifetime maintenance

Containers, power, network, ...

---

## Wait, is Linux a Microkernel?

<div class="center">

Yeah. :+1: :ninja:

</div>
