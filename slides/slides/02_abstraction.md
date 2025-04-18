---
title: Abstraction
---

# Abstraction

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2024, All rights reserved

</div>

---

# Abstraction

A simplified representation
- Omits details
- Encodes operations and abstractions over local state

Implemented in a **module/component**

---

## Abstraction Examples

- Filesystem
- Socket
- File descriptor
- Virtual Memory
- Thread
- Process
- Databases
- DOM
- ...

---

## Abstraction Properties

1. Interface
   - Programmer mechanism(s) to leverage abstraction
2. Implementation
   - Details of how the abstraction is provided
3. Resource interface
   - How the abstraction updates system resources

---

## Abstraction Properties

1. Interface - Programmer mechanism(s) to leverage abstraction
   - FS?
   - Process?
   - Virtual Memory?
   - DB?
2. Implementation - Details of how the abstraction is provided
3. Resource interface - How the abstraction updates system resources

---

## Abstraction Properties

1. Interface - Programmer mechanism(s) to leverage abstraction
2. Implementation - Details of how the abstraction is provided
3. Resource interface - How the abstraction updates system resources
   - FS?
   - DB?

```c []
struct sched_param sp;
sp.sched_priority = 10;
sched_setparam(getpid(), &sp);
```

---

Interface function specification:

>  sched_setparam()  sets  the  scheduling  parameters associated with the scheduling policy for the thread whose thread ID is specified  in  pid. If pid is zero, then the parameters of the calling thread are set.  The interpretation  of the argument param depends on the scheduling policy of the thread identified by pid. ...

Resource behavior specification:

> ...See sched(7) for a description of the scheduling policies supported under Linux.

---

# What Makes a Good Abstraction?

---

## Shallow vs. Deep Components

:one: How **wide** is the interface?

:two: How much detail does the component **hide**?

---

## Shallow vs. Deep Components

**Deep component** = higher value of:

$$\frac{\textsf{hidden implementation details}}{\textsf{interface width}}$$

Measure of *information hiding* $\to$ removing complexity from clients.

---

## Shallow vs. Deep Components

How **wide** is the interface *relative to* the hidden details?

- Narrow interface w/ significant implementation
  - Hides significant complexity
  - Organizes significant state

- Wide interface
  - Exposes more details about the implementation
  - Doesn't hide the complexity as much

**Deep components provide stronger abstraction**  <!-- .element: class="fragment" data-fragment-index="1" -->

---

## Shallow vs. Deep: Analogy

![cart ex.](resources/02_cart0.png)

---

## Shallow vs. Deep: Analogy

![cart ex.](resources/02_cart1.png)

---

## Shallow vs. Deep: Analogy

![cart ex.](resources/02_cart2.png)

---

## Shallow vs. Deep: Analogy

![cart ex.](resources/02_cart3.png)

---

## Deep Component Example

Unix Virtual File System Abstraction
- remarkably small API for a ton of functionality
- see the [xv6](https://github.com/gwu-cs-advos/xv6-riscv) source

```c []
int open(const char *pathname, int flags, mode_t mode);
off_t lseek(int fd, off_t offset, int whence);
ssize_t write(int fd, const void buf[], size_t count);
ssize_t read(int fd, void buf[], size_t count);
int close(int fd);
```

---

## Shallow Abstraction Example

<div class="center">

```mermaid
flowchart TD
	A(Client)
	B(Block Abstraction)
	C(Disk Driver)

	A --> B
	B --> C
```

</div>

Disk driver:
- Interface: block read/write
- Implementation: interrupts, DMA, registers

---


```c [1-3|5-14]
/* Block Abstraction */
int block_read(int offset, char *block);
int block_write(int offset, char *data);

/* Client Code */
void client(void)
{
	char block[4096];
	char *msg = "hello world";

	if (block_read(10, block) < 0) err();
	strncpy(block, msg, strlen(msg) + 1);
	if (block_write(10, block)) err();
}
```


Notes:
- What if we're preempted before the write, and someone else modifies the block?
- How does this interface differ from a SSD device driver?

---

## Abstraction Width $\neq$ Depth

The *width* of the `block_*` abstraction is very small.
- But provides little *abstraction*

Examples of similar abstractions:
- [`virtio_blk`](https://projectacrn.github.io/latest/developer-guides/hld/virtio-blk.html)
- Linux [Logical Volume Manager](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)) (LVM)
- [Exokernel](https://pdos.csail.mit.edu/papers/exo-sosp97/exo-sosp97.html) block interface

---

## Abstraction Width $\neq$ Depth

Interface to DBs?
- SQLite: `sqlite_exec(db_conn, "sql command...", ...)`
- Postgres: `PQexec(db_conn, "sql command...")`

Interface width $\neq$ number of functions <!-- .element: class="fragment" data-fragment-index="1" -->
- DB interface includes a full-fledged language  <!-- .element: class="fragment" data-fragment-index="1" -->

---

## Deep Components: Danger of Semantic Gap

Risk of deep components: The abstraction makes some actions impossible
- What if we want two files to share some blocks?
- What if we want to [share some of the page-tables](https://lwn.net/Articles/895217/) between processes?
- What if we want to [directly talk to devices](https://www.dpdk.org/) at user-level?

Semantic gap - If a task requires capabilities hidden and made inaccessible by the abstraction.  <!-- .element: class="fragment" data-fragment-index="1" -->

---

## Semantic Gap Example: `mmap`

`mmap` lets you directly access a file using virtual addresses and load/store.

Many DB systems have used `mmap` for fast access to DB files.
- Entire DB in memory, the system *will* run out of memory
- Kernel "paging" policies move memory$\to$disk
- Later, suffer page-fault and disk latency to get data

[DB semantic gap](https://db.cs.cmu.edu/papers/2022/cidr2022-p13-crotty.pdf): DB wants to manage what in the DB is in memory, `mmap` transparently removes that control.

---

## Semantic Gap Example: Networking

I want to talk to the Internet.

VFS doesn't have that thing.

- `open("/net/https://lobste.rs", ...)`?

---

## Deep Component - Networking Example

`socket`s augment the VFS API to enable network communication

- Streaming enables TCP/IP, without needing to know the details
- Can also use UDP if they require more per-packet control

<!-- --- -->

<!-- ## Shallow Components -->

<!-- Often expose *internal implementation details* in the interface -->

<!-- - Examples: -->
<!--   - an interface close to that of a data-structure - linked list API -->
<!--   - `block_*` interface w/ simple implementation -->
<!-- - *Couples component and client logic* -->
<!--   - Complicates component updates -->
<!--   - Moves complexity to client -->
<!--   - Defeats point of abstraction! -->

<!-- Leaky abstractions - an abstraction that reveals or requires client to have knowledge of implementation details.  <\!-- .element: class="fragment" data-fragment-index="1" -\-> -->

---

## [Leaky Abstractions](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/)

- Networking: latency spikes
  - TCP over wireless
- DBs: performance
  - manually specifying indices
  - denormalizing for performance
- CPU caches: performance
  - update data-structure granularity to cache-lines

Complexity is passed to the client

---

## Default Design Goal

1. Deep component (narrow interface, hide details)
2. Optimized interface for *common case* usage
   - For example, avoid whatever led to Java's `FileInputStream`, `BufferedInputStream`, I/O mess

---

## Extending Interfaces w/ Leaks

Deep component has a semantic gap?<br> $\to$ [extend interface](https://ieeexplore.ieee.org/document/183036) for more control where necessary<br> $\to$ often creates a leaky abstraction

---

## Deep Component with a Leaky Extension

```c []
int ioctl(int fd, unsigned long op, ...);
```

> Arguments,  returns, and semantics of ioctl() vary according to the device driver in question (the call is used as a catch-all for operations that don't cleanly fit the UNIX stream I/O model). - Manual page for `ioctl`

- Vague, explicitly "cuts through" the abstraction
- Specific, not general, only applicable to specific "files"

---

## File Access Leaky Extension

```c []
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

- Extension enables elision of system calls for file modifications
- Doesn't work on any `fd`, provides memory that doesn't behave like memory
- Have to understand what's happening in the abstraction
  - only works on files
  - Relies on eagerly created file contents (cannot `mmap` "files" in `/proc/`)

---

## Leaky Interface Extension Examples

Bridge the semantic gap, acknowledging leaky abstraction:

- `CREATE INDEX idxname ON table (col0, col1, ...)`
- `fsync` - to await data written to disk
- `O_DIRECT` - disable file's FS caching
- `setsockopt` - configure socket behavior
- `mlock` - memory should have mem-access latency
- `madvise` - `MADV_DONTNEED` "don't need this memory now, and when I access it next, it should be zeroed"
- `prefetch`, `clflush`, `mfence`, `movnti` - cache exception instructions

---

# Summary: Abstraction

---

```mermaid
mindmap
root((Abstraction))
	tool
		control complexity
	goal
		deep components
			narrow interface
			information hiding
			limited abstraction
			simpler to use
	challenges
		semantic gap
		leaky abstractions
```
