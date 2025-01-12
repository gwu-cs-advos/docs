---
title: Isolation
---

# System Structure and Isolation

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2025, All rights reserved

</div>

---

## System Structure

Where are *isolation boundaries* between components?

Where should controlled *communication* be allowed?

Where should *resource sharing* be allowed?

---

## Resource Isolation: Memory

*Memory isolation* - data-structures insulated from actions in other components
- $c_i$'s *set* of accessible pages: $m_i$
- Isolation: $m_i$ & $m_j$ are *disjoint* (i.e. $\forall_{i \neq j} m_i \cap m_j = \varnothing$)

---

## Resource Isolation: Abstract Resources

*Abstract resource isolation* - component abstractions provide "resources" separately for other components.
- One processes' file descriptors vs. another's
- Accessing only allowed files
- Abstraction might include "sharing" resources
- Similarly can be viewed as a set

---

## Resource Isolation: Control-flow Isolation

*Control-flow isolation* - each component should define and constraint its own allowable control flow.
- Control-Flow-Integrity (CFI)
- Kernel controls its entry points
  - Interrupt Descriptor Table (IDT) + `lidt` sensitive instruction
  - syscall entry point + `writemsr(MSR_LSTAR, syscall_entry);`

---

## Resource Isolation: Temporal Isolation

*Temporal isolation* - components get processing time when they need it
- won't focus on this till later

---

## Trust

Component $c_i$ *must trust* components it *depends on* ($\\{c_j, \ldots\\}$) to provide proper service
- A failure in $c_j$ can cause failures in $c_i$

```c
retval = foo(arg0, arg1);
```

What happens if
- `foo` has a logic error and returns incorrect values?
- `foo` segfaults?

---

## Trust

<div class="multicolumn">
<div>

Component $c_i$ *must trust* components it depends ($c_j$) on to provide proper service.
- Failure in $c_j$?

</div><div>

```mermaid
flowchart TB
	a["a :one: :wave:"]
	e["e :one:"]
	i["i :one: :two:"]
	h["h :two:"]
	c["c :two: :wave:"]
	f["f :two:"]
	a --> e
	b --> e
	c --> f
	d --> f
	e --> i
	g --> h
	f --> h
	k --> i
	h --> i
```

</div></div>

---

## Trust

<div class="multicolumn">
<div>

Component $c_i$ *must trust* components it depends ($c_j$) on to provide proper service.
- Failure in $c_j$?

</div><div>

```mermaid
flowchart TB

J(["\nNTP\nclient :two:\n\n"])
H(["\nNet\nstack :two:\n\n"])
I(["\nNIC device\ndriver :one: :two:\n\n"])
A(["\nFilesystem\n\n"])
B(["\nSSD\ndriver\n\n"])
C(["\nHTTP\nwebserver\n\n"])
D(["\nRouting :one:\n\n"])
E(["\nShared\nmemory :one:\n\n"])

J-->H
H-->I
C-->H
C-->A
A-->B
D-->I
D-->E
C-->E
```

</div></div>

---

## But it isn't that simple...

<div class="multicolumn">
<div>

Failure in the *Filesystem*?
- What if the "Filesystem" component isn't isolated from the "NIC device driver"? <!-- .element: class="fragment" data-fragment-index="1" -->

</div><div>

```mermaid
flowchart TB

J(["\nNTP\nclient :two:\n\n"])
H(["\nNet\nstack :two:\n\n"])
I(["\nNIC device\ndriver :one: :two:\n\n"])
A(["\nFilesystem :boom:\n\n"])
B(["\nSSD\ndriver\n\n"])
C(["\nHTTP\nwebserver\n\n"])
D(["\nRouting :one:\n\n"])
E(["\nShared\nmemory :one:\n\n"])

J-->H
H-->I
C-->H
C-->A
A-->B
D-->I
D-->E
C-->E
```

</div></div>

---

## Complicated

<div class="multicolumn">
<div>

App just talks over the network.
- Failure in *Block Cache*?

</div><div>

```mermaid
block-beta
	columns 3
	block p0["app:wave:"] end
	block p1["app"] end
	block p2["app"] end
	block:KERN:3
		columns 3
		space:1
		vfs["VFS interface"]
		space:1

	    pipe["Pipes"]
		fs["File System"]
		sock["Sockets"]
		space:1
		cache["Block Cache :boom:"]
		ip["Net Stack"]
	    sched["Scheduler"]
		driver["SSD driver"]
	    nic["NIC Driver"]
	end
```

</div></div>

---

## Trust is Viral

- $c_i$ functionally depends on $d_i = \\{c_j, \ldots\\}$
- $c_i$ has access to resources $r_i$ (memory, files, etc...)
- $c_i$ must trust $t_i = \\{c_j,\ldots\\}$
- $t_i = t_i \cup \\{c_k | r_j \cap r_k \neq \varnothing, c_j \in t_i + c_i\\}$

---

```mermaid
block-beta
	columns 3
	block p0["app:wave:"] end
	block p1["app"] end
	block p2["app"] end
	block:KERN:3
		columns 3
		space:1
		vfs["VFS interface"]
		space:1

	    pipe["Pipes"]
		fs["File System"]
		sock["Sockets"]
		space:1
		cache["Block Cache :boom:"]
		ip["Net Stack"]
	    sched["Scheduler"]
		driver["SSD driver"]
	    nic["NIC Driver"]
	end
```

---

```mermaid
block-beta
	columns 3
	block p0["app:wave:"] end
	block p1["app"] end
	block p2["app"] end
	block:KERN:3
		columns 3
		space:1
		vfs["VFS interface:boom:"]
		space:1

	    pipe["Pipes:boom:"]
		fs["File System:boom:"]
		sock["Sockets:boom:"]
		space:1
		cache["Block Cache :boom:"]
		ip["Net Stack:boom:"]
	    sched["Scheduler:boom:"]
		driver["SSD driver:boom:"]
	    nic["NIC Driver:boom:"]
	end
```

---

```mermaid
block-beta
	columns 3
	block p0["app:boom:"] end
	block p1["app:boom:"] end
	block p2["app:boom:"] end
	block:KERN:3
		columns 3
		space:1
		vfs["VFS interface:boom:"]
		space:1

	    pipe["Pipes:boom:"]
		fs["File System:boom:"]
		sock["Sockets:boom:"]
		space:1
		cache["Block Cache :boom:"]
		ip["Net Stack:boom:"]
	    sched["Scheduler:boom:"]
		driver["SSD driver:boom:"]
	    nic["NIC Driver:boom:"]
	end
```

---

## Isolation

Isolation matters when we must understand
- reliability properties
- security properties
- multi-tenancy properties

---

## Memory Isolation

Mechanism: page-tables

Questions:
- Which components should share a page-table?
- What component controls the page-tables?
- Which page-tables do they use?

---

## Abstract Resource Examples

- File
- Directory
- Pipe
- Socket
- Lock
- Condition Variables
- Threads
- Processes
- Virtual Memory Page

---

## Abstract Resource Isolation

In what ways are UNIX Processes isolated from each other?

In what ways are they not?

What abstract resources are involved?

---

# How to Think about Isolation

---

## Vertical Isolation

<div class="multicolumn">
<div>

Enforce *information hiding/encapsulation* of abstractions in lower layers.
Mechanisms:
- dual-mode isolation, syscalls, exceptions
- HW virt., VMEXIT, VMRESUME

</div>
<div>

```mermaid
block-beta
	columns 1
	block:UserLvl
		columns 1
		p0(["Application"])
		lib(["Library"])
	end
	block:VMKern
		kern(["Kernel"])
	end
	block:Hypervisor
		columns 1
		VMM(["VMM"])
		space
		HostKern(["Host Kernel"])
	end
	p0 --> lib
	lib --"syscall"--> kern
	kern --"hypercall"--> VMM
	VMM --> HostKern
```

</div></div>

---

<div class="multicolumn">
<div>

## Vertical Isolation + Trust

```mermaid
flowchart TB
subgraph _
a["<i>c<sub>a</sub></i>"]
end
subgraph __
b["<i>c<sub>b</sub></i>"]
end

a-->b
```

</div><div>

Implied relations

- $c_b \in t_a$
- $c_a \not\in t_b$

Implications:
- Functions in $c_b$ *must not* trust the validity of inputs from $c_a$
- Failures in $c_a$ should not impact $c_b$

</div></div>

---

## Horizontal Isolation

<div class="multicolumn">
<div>

Isolation between *siblings* (generally: *subtrees*)
- page-tables, ASIDs, write to `cr3`
- multi-user, -process, -VM

</div><div>

```mermaid
block-beta
	columns 2
	block:VM0
		columns 2
		block:U0 p0["app"] end
		block:U1 p1["app"] end
		k0["kernel"]:2
	end
	block:VM1
		columns 2
		block:U2 p2["app"] end
		block:U3 p3["app"] end
		k1["kernel"]:2
	end
	space
	block:VMM:2
		columns 1
		vmm["VMM"]
		hv["hypervisor"]
	end
```

</div></div>

---

## Horizontal Isolation (HI)

Requires Vertical Isolation (VI)
- HI must be implemented with abstractions
- The system must ensure the integrity of those abstractions
- VI ensures this integrity

---

<div class="multicolumn">
<div>

## Horizontal Isolation + Trust

```mermaid
flowchart TB
subgraph _
a["<i>c<sub>a</sub></i>"]
end
subgraph __
b["<i>c<sub>b</sub></i>"]
end
subgraph ___
c["<i>c<sub>k</sub></i>"]
end

a-->c
b-->c
```

</div><div>

Implied relations
- $c_k \in t_a$, $c_a \not\in t_k$
- $c_k \in t_b$, $c_b \not\in t_k$

$\to c_a \not\in c_b \wedge c_b \not\in c_a$

Implications:
- $c_k$ must separate resources accessed for $c_a$ and $c_b$

</div></div>

---

## Mitigating Trust

What if we have a dependency, but want to be tolerant to faults in it?

> Imagine: we don't want to trust
> - your DB in a tiered web service, or
> - the Linux kernel?

How can we do this?

---

## Isolation: Resource Abstractions

The core abstraction being provided
- File
- Process
- Virtual Memory Page

Clients ask for resources & perform operations on them

---

## Isolation: Resource Namespaces

Clients use *names* to reference their resources
- pointer
- integer
- path

Why one or the other?

---

# Resource Naming & Binding

---

## Isolation: Resource Naming

Paths as the name of resources

```c []
int creat(const char *pathname, mode_t mode);
int unlink(const char *pathname);
int stat(const char *restrict pathname, struct stat *restrict statbuf);

int link(const char *oldpath, const char *newpath);
int rename(const char *oldpath, const char *newpath);

int rmdir(const char *pathname);
int chdir(const char *path);
int mkdir(const char *pathname, mode_t mode);
```

---

## Isolation: Resource Naming

File descriptors

```c []
off_t lseek(int fd, off_t offset, int whence);
ssize_t write(int fd, const void buf[], size_t count);
ssize_t read(int fd, void buf[], size_t count);
int close(int fd);
```

---

## Isolation: Resource Naming

```mermaid
flowchart TB

subgraph process
a1["application"]
p20>"descriptor"]
p21>"path"]
end

subgraph kernel
	r20[\"resource"\]
	r21[\"resource"\]
end

a1-->p20
a1-->p21
p20-->r20
p21-->r21
```

---

## Isolation: Naming w/ Pointers

```c []
/* pthread pointer to struct, contains integer */
int pthread_create(pthread_t *restrict thread, const pthread_attr_t *restrict attr, void *(*start_routine)(void *), void *restrict arg);
/* pid_t = int */
pid_t fork(void);
/* generalization of both, retval = identifier*/
int clone(int (*fn)(void *), void *stack, int flags, void *arg, ...);
```

- [`struct pthread`](https://git.musl-libc.org/cgit/musl/tree/src/internal/pthread_impl.h#n18) identified by pointer, contains integers to pass to the kernel.
- `fork` identifies process by integer
- `clone` is called by both, returns resource name as id

---

## Isolation: Naming w/ Pointers

```mermaid
flowchart TB

subgraph process
a["application"]
p0>"ptr"]
p1>"ptr"]
subgraph library
	s0[\"struct"\]
	s1[\"struct"\]
	d0>"descriptor"]
	d1>"descriptor"]
end
end

subgraph kernel
	r0[\"resource"\]
	r1[\"resource"\]
end

a-->p0
a-->p1
p0-->s0
p1-->s1
s0-->d0
s1-->d1
d0-->r0
d1-->r1
```

---

## Isolation: Naming w/ Pointers

Why not use pointers to name kernel resources?

---

## Isolation: Names w/ Pointers

Pointers as names that span user/kernel
```c []
/* name is pointer into library */
int pthread_mutex_lock(pthread_mutex_t *mutex);
/* address used as name for futex */
long syscall(SYS_futex, uint32_t *uaddr, int futex_op, uint32_t val, const struct timespec *timeout, uint32_t *uaddr2, uint32_t val3);
```

- kernel must translate (using a HT) from *pointer* into *kernel resource* - why not simply dereference?
- A *exceptional/rare* API

---

## Isolation: Name/Resource Binding

```c []
/* new path binding for a file resource */
int creat(const char *pathname, mode_t mode);
/* new descriptor binding for resource at path */
int open(const char *pathname, int flags, mode_t mode);
DIR *opendir(const char *name);
/* new descriptor binding from network namespace */
int socket(int domain, int type, int protocol);
/* new descriptor binding from descriptor */
int dup(int oldfd);
int dup2(int oldfd, int newfd);
int accept(int sockfd, struct sockaddr * restrict addr, socklen_t * restrict addrlen);
/* Virtual addr binding to memory*/
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

---

## Isolation: Name/Resource Binding

```mermaid
flowchart TB

subgraph process
a1["application"]
p20>"descriptor"]
p21>"path"]
end

subgraph kernel
	b[["<i>resolution</i>(name) → resource"]]
	r20[\"resource"\]
	r21[\"resource"\]
end

a1-->p20
a1-->p21
p20-->b
b-->r20
p21-->b
b-->r21
```

---

## Horizontal Isolation: Name/Resource Binding

```mermaid
flowchart TB

subgraph process
a1["application"]
p20>"descriptor"]
p21>"path"]
end

subgraph kernel
	b[["<i>resolution</i>(<font color=red>client_pid</font>, name) → resource"]]
	r20[\"resource"\]
	r21[\"resource"\]
end

a1-->p20
a1-->p21
p20-->b
b-->r20
p21-->b
b-->r21
```

---

## Binding & Resolution

Binding:
- file descriptor $\leftrightarrow$ struct file

Resolution:
- per-process file descriptor table

---

```c []
/* From xv6 */
static int
argfd(int n, int *pfd, struct file **pf)
{
	int          fd;
	struct file *f;

	argint(n, &fd);
	if (fd < 0 || fd >= NOFILE || (f = myproc()->ofile[fd]) == 0) return -1;
	if (pfd) *pfd = fd;
	if (pf) *pf = f;
	return 0;
}
```

---

```c [4-20|9,15|40-46]
/* From Linux */

/* include/linux/fdtable.h */
struct files_struct {
	atomic_t count;
	bool resize_in_progress;
	wait_queue_head_t resize_wait;

	struct fdtable __rcu *fdt;
	struct fdtable fdtab;
	/* ... */
}
struct fdtable {
	unsigned int max_fds;
	struct file __rcu **fd;      /* current fd array */
	unsigned long *close_on_exec;
	unsigned long *open_fds;
	unsigned long *full_fds_bits;
	struct rcu_head rcu;
};

/* fs/file.c: called from read_write.c:ksys_read(...) */
struct file *fget(unsigned int fd)
{
	/* eventually does... */
	struct files_struct *files = current->files; /*-ish */
	struct file *file;
	struct fdtable *fdt = rcu_dereference_raw(files->fdt);
	struct file __rcu **fdentry;
	unsigned long nospec_mask;

	/* Mask is a 0 for invalid fd's, ~0 for valid ones */
	nospec_mask = array_index_mask_nospec(fd, fdt->max_fds);

	/*
	 * fdentry points to the 'fd' offset, or fdt->fd[0].
	 * Loading from fdt->fd[0] is always safe, because the
	 * array always exists.
	 */
	fdentry = fdt->fd + (fd & nospec_mask);

	/* Do the load, then mask any invalid result */
	file = rcu_dereference_raw(*fdentry);
	/* ... */

	return file;
}
```

Notes:
The kernel code for this is pretty intractable.
The high-level description of the [`CLASS`](https://lwn.net/Articles/934679/) macro helps.

---

## Isolation Defaults

*Security by default:*
1. Descriptor (int) names
2. Partitioned namespaces

---

# Shared vs. Partitioned Namespaces

---

## Shared vs. Partitioned Namespaces

Partitioned:
- File descriptors: per-process resolution, $fdtable(pid, fd) \to file$

Shared:
- Hierarchical paths: global resolution w/ permission checking, $fs(uid, path) \to dentry$

Trade-offs?

Notes:

Downsides of global:
- Ambient authority - permission checking among a huge amount of logic
- Sharing by default = no isolation = necessary hierarchical trust

Upsides:
- Sharing by default, enabling
  1. enabling efficient allocation/use of resources, and
  2. ease of sharing and coordination among users/processes

---

## Example: Linux Namespace

[Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces):
1. Mount points & filesystem
2. Process identifiers: `pid`s
3. Network namespace (IP, routing, firewall rules, sockets, ...)
4. SysV IPC namespace
5. UTS - host and domain
6. User/group ids (id = 0)
7. `cgroups` - ids to reference the resource group
8. Time namespace - system times

---

## Example: Containers

Containers provide APIs/facilities to *partition* each namespace

For example:
- two processes in separate containers can have the same `pid`.

Docker:
- partition the namespaces to make each container feel independent

---

## Example: Containers

Do containers provide isolation?
- Kernel still provides name bindings to resources - no modification to vertical isolation
- Disjoint names across containers enable restricted resource sharing - increased horizontal isolation

---

## Example: Capabilities

Each component has a *capability table*:
- Component $c_i$: $captbl_i[cap] \to r_a$
- *All* syscalls request to perform an operation on a capability-indexed resource
- $syscall(cap, op, ...): captbl_i[cap] \to r_a, op(r_a)$

Single namespace: capabilities
- Clearly isolate component
- Can *only* access resources in capability table

---

## Isolation Defaults

*Security by default:*
1. Descriptor (int) names
2. Partitioned namespaces

---

## Sharing w/ Partitioned Namespaces

Is this possible?
- Need some mechanism to send resources between separate namespaces

---

## UNIX Domain Sockets

```mermaid
flowchart TB

subgraph _
a["client"]
end
subgraph __
c["server"]
end

subgraph kernel
d>"fd"]
f>"fd"]
e>"fd"]
b[\"UNIX Domain Socket"\]
g[\"resource"\]
end

a-->d-->b
a-->f-->g
c-->e-->b
```

---

## UNIX Domain Sockets

```mermaid
flowchart TB

subgraph _
a["client"]
end
subgraph __
c["server"]
end

subgraph kernel
d>"fd"]
f>"fd"]
e>"fd"]
h>"fd"]
b[\"UNIX Domain Socket"\]
g[\"resource"\]
end

c-.->h-.->g
a-->d-->b
c-->e-->b
a-->f-->g
```

---

## Sharing with UNIX Domain Sockets

- Uses `SCM_RIGHTS` (example later) to pass the rights to a `fd`
- Overview of [UNIX Domain sockets](https://gwu-cs-advos.github.io/sysprog/#unix-domain-sockets)
- Sending/receiving file-descriptors and credentials via IPC
  - Sending: https://man7.org/tlpi/code/online/dist/sockets/scm_multi_send.c.html
  - Receiving: https://man7.org/tlpi/code/online/dist/sockets/scm_multi_recv.c.html

---

## Resources

[The Linux Programming Interface](https://man7.org/tlpi/code/index.html)
- Great resource of Linux programs using multiple system features
- Simple “ping/pong” (send and receive) communication between client and server:
  - [Client](https://man7.org/tlpi/code/online/dist/sockets/ud_ucase_cl.c.html) doing send/receive
  - [Server](https://man7.org/tlpi/code/online/dist/sockets/ud_ucase_sv.c.html) doing receive/send

---

## Delegating Resources

A component can provide access to its resources to another, communicating component

But how do we get the `fd` to the domain socket?

---

## Delegating Resources

Shared namespace to bootstrap
- Necessary at some point to "bootstrap" sharing
- Create domain socket, get fd to domain socket from `fork`, etc...

---

# Client Authentication

---

## Client Authentication

A server component must be provided *by the system* with a trustworthy *client identity*.
- *Access rights* - Which resources are accessible?
- *Namespaces* - Which namespace to use?
- *Accounting* - Accounting for resource allocations?

---

## UNIX

1. **Per-client communication channels** - differentiate requests for each client
2. **UID-based authentication** - determine the UID of the client of the channel

---

## UNIX Domain Socket Per-Client

```mermaid
flowchart TB

subgraph _
a["client"]
end
subgraph __
c["server"]
end

subgraph kernel
d>"fd"]
e>"fd"]
b[\"Named Domain Socket"\]
end

a-->d-->b
c-->e-->b
```

---

## UNIX Domain Socket Per-Client

```mermaid
flowchart TB

subgraph _
a["client"]
end
subgraph __
c["server"]
end

subgraph kernel
d>"fd"]
e>"fd"]
g>"fd"]
b[\"Named Domain Socket"\]
f[\"Domain Socket"\]
end

a-->d-->f
c-->e-->b
c-->g-->f
```

---

## UNIX Domain Sockets: Per-client

```mermaid
sequenceDiagram
	autonumber
    server-->>kernel: accept(srv_fd...)
    client->>kernel: send_msg(cli_fd,..., fd, ...)
	kernel-->>server: newfd = accept(...)
    server-->>kernel: recv_msg(newfd, ...)
	kernel-->>server: { srv_fd, ... } = recv_msg(newfd, ...)
	server-->>kernel: send_msg(newfd, "hello world")
	kernel-->>client: "hello world" = recv_msg(newfd, ...)
```

---

## Authentication: Who are We Talking To?

UNIX Domain Sockets provide facilities for client to send credentials (via `SCM_CREDENTIALS`), and server to receive them.

Example [client](https://man7.org/tlpi/code/online/dist/sockets/ud_ucase_cl.c.html) and [server](https://man7.org/tlpi/code/online/dist/sockets/ud_ucase_sv.c.html).

Very common. Postgres does this:
- Get [process credentials](https://doxygen.postgresql.org/getpeereid_8c_source.html)
- Domain socket-based [communication](https://doxygen.postgresql.org/pqcomm_8c_source.html)

---

```c [15-17|23-34|42-60|63-66]
/* scm_multi_send.c from TLPI @ https://man7.org/tlpi/code/online/dist/sockets/scm_multi_send.c.html

   Used in conjunction with scm_multi_recv.c to demonstrate passing of
   ancillary data containing multiple 'msghdr' structures on a UNIX
   domain socket.

   This program sends ancillary data consisting of two blocks.  One block
   contains process credentials (SCM_CREDENTIALS) and the other contains
   one or more file descriptors (SCM_RIGHTS).

*/
int
main(int argc, char *argv[])
{
	pid_t pid = getpid();
    uid_t uid = getuid();
    gid_t gid = getgid();

    /* Allocate a buffer of suitable size to hold the ancillary data.
       This buffer is in reality treated as a 'struct cmsghdr',
       and so needs to be suitably aligned: malloc() provides a block
       with suitable alignment. */
    size_t fdAllocSize = sizeof(int);
    size_t controlMsgSize = CMSG_SPACE(fdAllocSize) +
                     CMSG_SPACE(sizeof(struct ucred));
    char *controlMsg = malloc(controlMsgSize);
    msgh.msg_control = controlMsg;
    msgh.msg_controllen = controlMsgSize;

    /* Set message header to describe the ancillary data that
       we want to send. First, the file descriptor list */
    struct cmsghdr *cmsgp = CMSG_FIRSTHDR(&msgh);
    cmsgp->cmsg_level = SOL_SOCKET;
    cmsgp->cmsg_type = SCM_RIGHTS;

    /* The ancillary message must include space for the required number
       of file descriptors */
    cmsgp->cmsg_len = CMSG_LEN(fdAllocSize);

    /* Open files named on the command line, and copy the resulting block of
       file descriptors into the data field of the ancillary message */
    int *fdList = malloc(fdAllocSize);
	/* An FD to send */
	fdList[0] = open(argv[1], O_RDONLY);
    memcpy(CMSG_DATA(cmsgp), fdList, fdAllocSize);

    /* Next, the credentials */
    cmsgp = CMSG_NXTHDR(&msgh, cmsgp);
    cmsgp->cmsg_level = SOL_SOCKET;
    cmsgp->cmsg_type = SCM_CREDENTIALS;

    /* The ancillary message must include space for a 'struct ucred' */
    cmsgp->cmsg_len = CMSG_LEN(sizeof(struct ucred));
    /* Initialize the credentials that are to be placed in the ancillary data */
    struct ucred creds;
    creds.pid = pid;
    creds.uid = uid;
    creds.gid = gid;
    /* Copy credentials to the data area of this ancillary message block */
    memcpy(CMSG_DATA(cmsgp), &creds, sizeof(struct ucred));

    /* Connect to the peer socket using a utility function */
    int sfd = unixConnect("./named_domain_socket_file", SOCK_STREAM);

    /* Send the data plus ancillary data */
    ssize_t ns = sendmsg(sfd, &msgh, 0);
	sleep(1000);
	printf("sendmsg() returned %zd\n", ns);

    exit(EXIT_SUCCESS);
}
```

---

```c [21-23|34-44|57-70|74-84]
/* scm_multi_recv.c from TLPI @ https://man7.org/tlpi/code/online/dist/sockets/scm_multi_recv.c.html

   Used in conjunction with scm_multi_send.c to demonstrate passing of
   ancillary data containing multiple 'msghdr' structures on a UNIX
   domain socket.
*/
int
main(int argc, char *argv[])
{
    /* Allocate a buffer of suitable size to hold the ancillary data.
       This buffer is in reality treated as a 'struct cmsghdr',
       and so needs to be suitably aligned: malloc() provides a block
       with suitable alignment. */
    size_t controlMsgSize = CMSG_SPACE(sizeof(int[MAX_FDS])) +
                            CMSG_SPACE(sizeof(struct ucred));
    char *controlMsg = malloc(controlMsgSize);

	/* Create socket bound to a well-known address. In the case where
       we are using stream sockets, also make the socket a listening
       socket and accept a connection on the socket. */
	int lfd = unixBind("./named_domain_socket_file", SOCK_STREAM);
    if (listen(lfd, 5) == -1)
	int sfd = accept(lfd, NULL, NULL);

	/* We must set the SO_PASSCRED socket option in order to receive
       credentials */
    int optval = 1;
    if (setsockopt(sfd, SOL_SOCKET, SO_PASSCRED, &optval, sizeof(optval)) == -1)
        errExit("setsockopt");

    /* The 'msg_name' field can be set to point to a buffer where the
       kernel will place the address of the peer socket. However, we don't
       need the address of the peer, so we set this field to NULL. */
    struct msghdr msgh;
    msgh.msg_name = NULL;
    msgh.msg_namelen = 0;
    msgh.msg_iovlen = 0;

    /* Set 'msgh' fields to describe the ancillary data buffer. */
    msgh.msg_control = controlMsg;
    msgh.msg_controllen = controlMsgSize;

    /* Receive real plus ancillary data */
    ssize_t nr = recvmsg(sfd, &msgh, 0);
    printf("recvmsg() returned %zd\n", nr);

	/* Walk through the series of headers in the ancillary data */
    for (struct cmsghdr *cmsgp = CMSG_FIRSTHDR(&msgh);
		 cmsgp != NULL;
         cmsgp = CMSG_NXTHDR(&msgh, cmsgp)) {
        printf("cmsg_len: %ld\n", (long) cmsgp->cmsg_len);

        /* Check that 'cmsg_level' is as expected */
        if (cmsgp->cmsg_level != SOL_SOCKET) fatal("cmsg_level != SOL_SOCKET");

        switch (cmsgp->cmsg_type) {
        case SCM_RIGHTS:        /* Header containing file descriptors */
            /* The number of file descriptors is the size of the control
               message block minus the size that would be allocated for
               a zero-length data block (i.e., the size of the 'cmsghdr'
               structure plus padding), divided by the size of a file
               descriptor */
            int fdCnt = (cmsgp->cmsg_len - CMSG_LEN(0)) / sizeof(int);

            /* Allocate an array to hold the received file descriptors,
               and copy file descriptors from cmsg into array */
            int *fdList;
            size_t fdAllocSize = sizeof(int) * fdCnt;
            fdList = malloc(fdAllocSize);
            memcpy(fdList, CMSG_DATA(cmsgp), fdAllocSize);

	        /* ...use fdList[j] to access file descriptors */
            break;
        case SCM_CREDENTIALS:   /* Header containing credentials */
            /* Check validity of the 'cmsghdr' */
            if (cmsgp->cmsg_len != CMSG_LEN(sizeof(struct ucred)))
                fatal("cmsg data has incorrect size");

            /* The data in this control message block is a 'struct ucred' */
            struct ucred creds;
            memcpy(&creds, CMSG_DATA(cmsgp), sizeof(struct ucred));
            printf("SCM_CREDENTIALS: pid=%ld, uid=%ld, gid=%ld\n",
                        (long) creds.pid, (long) creds.uid, (long) creds.gid);
            break;
        default:
            fatal("Bad cmsg_type (%d)", cmsgp->cmsg_type);
        }
    }

    exit(EXIT_SUCCESS);
}
```

---

## Controlling Communication Topology

```mermaid
flowchart TB

subgraph _
a["client 0"]
end
subgraph ___
f["client 1"]
end

subgraph __
c["server"]
end

subgraph kernel
d>"fd"]
e>"fd"]
g>"fd"]
b[\"Named Domain Socket"\]
end

a-->d-->b
f-->g-->b
c-->e-->b
```

---

## Controlling Communication Topology

```mermaid
flowchart TB

subgraph _
a["client 0"]
end
subgraph ___
f["client 1"]
end

subgraph __
c["server"]
end

subgraph kernel
d>"fd"]
e>"fd"]
g>"fd"]
j>"fd"]
k>"fd*"]
i[\"Domain Socket"\]
h[\"Domain Socket"\]
b[\"Named Domain Socket"\]
end

a-->d-->i
f-->g-->h
c-->e-->b
c-.->j-.->i
c-.->k-.->h
```

---

## Controlling Communication Topology

```mermaid
flowchart TB

subgraph _
a["client 0"]
end
subgraph ___
f["client 1"]
end

subgraph __
c["server"]
end

subgraph kernel
d>"fd"]
e>"fd"]
g>"fd"]
j>"fd"]
k>"fd*"]
l>"fd*"]
i[\"Domain Socket"\]
h[\"Domain Socket"\]
b[\"Named Domain Socket"\]
end

a-->d-->i
a-->l-->h
f-->g-->h
c-->e-->b
c-->j-->i
c-->k-->h
```

---

## Controlling Communication Topology

```mermaid
flowchart TB

subgraph _
a["client 0"]
end
subgraph ___
f["client 1"]
end

subgraph __
c["server"]
end

subgraph kernel
d>"fd"]
e>"fd"]
g>"<font color="red">fd</font>"]
j>"fd"]
l>"<font color="red">fd</font>"]
i[\"Domain Socket"\]
h[\"<font color="red">Domain Socket</font>"\]
b[\"Named Domain Socket"\]
end

a-->d-->i
a-->l-->h
f-->g-->h
c-->e-->b
c-->j-->i
```

---

## Nameserver

Server component provides $\textsf{resolve}(name) \to resource$
- client requests name
- server sends `fd` to resource

DNS, DHCP, systemd, binder

---

## Nameserver

Server enables
1. selective access to its resources (*abstraction*), and/or
   - Vertical isolation between clients and server
   - Horizontal isolation between clients
2. restricted inter-client access to client's resources (*enable client abstraction*).
   - Vertical isolation *between clients*

We're creating our own system structure!

---

## Nameserver

What have we done?
- Existing UNIX system structure (system relationships, isolation properties) $\to$
- Our own system system structure (relationships and isolation properties)

Logical conclusion:
- Android
- `systemd`

---

# Summary: Isolation

---

```mermaid
mindmap
root ((isolation))
	dependencies
		trust
		fault tolerance
	isolation classification
		horizontal
		vertical
	namespaces
		descriptor-based
		partitioned
		shared
		resources
			binding
			resolution
	client authentication
	implementation
		UNIX Domain Sockets
		vertical isolation
		nameserver
			horizontal isolation
```