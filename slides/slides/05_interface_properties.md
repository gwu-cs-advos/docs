---
title: Interface Properties
---

# Interface Properties

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2025, All rights reserved

</div>

---

## Interface & Component Design

- How can we think about interface design?
- What do they mean about component design?
- What are the *systems considerations* we have to make?

---

## Virtualization

Virtualizing an interface
- Providing an interface $A$ similar to another $A'$
- ...where $A'$ depends on $A$
- ...thus providing a higher-level abstraction
- Example: VMs vs. hardware

Virtualizing a resource
- Provide a more abstract version of that resource
- CPU $\to$ thread
- physical memory$\to$virtual memory
- interrupt$\to$signal

---

## Virtualization

Virtual resource namespace
- Separate namespace for virtualized resources
- Component providing virtualization: $m(r_v)\to r_p$

Examples:
- Virt Mem: page-tables virtualize phys mem
- VMs:
  - virt phys mem w/ extended page-tables
  - virt cores: vCPUs
- Containers: virt all namespaces
  - FS: new `/` to FS

---

## Liveness and Data Placement in Interfaces

*Liveness* - Who is in charge of freeing a resource?
- who is the *owner* of the resource?
- who *borrows* a resource during a call?

*Data placement* - When data addresses are passed between components,
- what names (addresses) are used,
- who allocates the names, and
- who has access to the names?

---

## Liveness/Data Placement in Interfaces

Implicit in *every C interface*

:one: Client allocates, server borrows
```c []
char *buf = malloc(BUFSZ);
server_fn(buf, BUFSZ);
free(buf);
```
Client determines data location.

---

## Liveness/Data Placement in Interfaces

:two: Client allocates, passes ownership to server
```c []
char *buf = malloc(BUFSZ);
server_fn(buf, BUFSZ);
```
Client determines data location.

---

## Liveness/Data Placement in Interfaces

:three: Server allocates, passes ownership to client
```c []
char *buf;
int sz;
server_fn(&buf, &sz);
free(buf);
```
Server determines data location.

---

```c [1,2|1,3-9]
/* :one: Client allocates, provites data location, kernel borrows */
ssize_t read(int fd, void *buf, size_t count);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
char *str = "hello world\n";
ssize_t nwritten;
struct iovec iov[1];
iov[0].iov_base = str;
iov[0].iov_len = strlen(str);
nwritten = writev(STDOUT_FILENO, iov, 1);
```

---

```c++ [2,5|3,5-14|11]
/* :three: Server allocates, passes ownership to client */
int dmtr::io_queue_api::pop(dmtr_qtoken_t &qtok_out, int qd);
int dmtr::io_queue_api::poll(dmtr_qresult_t &qr_out, dmtr_qtoken_t qt);
/* where ... */
typedef uint64_t dmtr_qtoken_t;
typedef struct dmtr_qresult {
    enum dmtr_opcode qr_opcode;
    int qr_qd;
    dmtr_qtoken_t qr_qt;
    union {
        dmtr_sgarray_t sga;
        dmtr_accept_result_t ares;
    } qr_value;
} dmtr_qresult_t;
```

-v-

From the [demikernel](https://www.microsoft.com/en-us/research/project/demikernel/) [source](https://github.com/gwu-cs-advos/demikernel) and these examples from the [types](https://github.com/gwu-cs-advos/demikernel/blob/master/include/dmtr/types.h).

---

## Shared Memory Allocation

:one:, :two:, and :three: across isolated components?

---

## Shared Memory Allocation

[Shared memory w/ bitmap allocation](https://github.com/gwsystems/composite/blob/main/src/components/lib/shm_bm/doc.md) in [Composite](https://github.com/gwsystems/composite/)
- slab allocator in shared memory
- Memory object references that can be shared across memory virtual address spaces
- namespace:
  - `shm_bm_t` - the shared memory pool allocated from
  - `shm_objid_t` - the object id within a specific pool

---

# Concurrency

---

## Concurrency in Implementations

Component state relevant for $c_i$ can be updated by
1. client requests from $c_i$,
2. client requests from $c_j, i \neq j$, and/or
3. interactions with lower-level (server) components.

Example: `read` from a file

---

## Concurrency in Implementations

Mechanisms:
- *Correctness* - locks, lock-free data-structures
- *Thread interactions* - condition variables, semaphores

Example: KV-store where you can wait for a specific key

---

```c [1-34|13,26|11,18|13-16|26-29]
mutex_t lock = MUTEX_INIT;
condvar_t cv = CV_INIT;
volatile int awaiting_value = 0;
struct hash_table ht;

value_t
kv_get(key_t k)
{
	value_t v = VALUE_NULL;

	mutex_lock(&lock);
	while (1) {
		v = ht_get(&ht, k);
		if (v != VALUE_NULL) break;
		awaiting_value = 1;
		condvar_wait(&cv, &lock);
	}
	mutex_unlock(&lock);

	return v;
}

kv_put(key_t k, value_t v)
{
	mutex_lock(&lock);
	ht_put(&ht, k, v);
	if (awaiting_value) {
		awaiting_value = 0;
		condvar_signal_broadcast(&cv);
	}
	mutex_unlock(&lock);

	return v;
}
```

---

## Concurrency in Implementations

Implementation must consider:
- different clients don't want to trust each other - horizontal isolation
- avoid cross-client blocking/progress?
- avoid cross-client serialization/lock delays?
- shared namespaces/shared resources makes this hard
  - example with separate namespaces

What are the *interface properties* that enable this?

---

```c [1-10|16|18-25]
struct client_data {
	mutex_t lock;
	condvar_t cv;
	volatile int awaiting_value;
	struct hash_table ht;
} clients[CLIENT_NUM];
/*
 * We're "cheating" here as we know we can
 * access `clients` without a lock to simplify.
 */

value_t
kv_get(key_t k)
{
	value_t v = VALUE_NULL;
	struct client_data *c = clients[client_id()];

	mutex_lock(&c->lock);
	while (1) {
		v = ht_get(&c->ht, k);
		if (v != VALUE_NULL) break;
		c->awaiting_value = 1;
		condvar_wait(&c->cv, &c->lock);
	}
	mutex_unlock(&c->lock);

	return v;
}

void
kv_put(key_t k, value_t v)
{
	struct client_data *c = clients[client_id()];

	mutex_lock(&c->lock);
	ht_put(&c->ht, k, v);
	if (c->awaiting_value) {
		c->awaiting_value = 0;
		condvar_signal_broadcast(&c->cv);
	}
	mutex_unlock(&c->lock);

	return v;
}
```

---

## Blocking vs. Non-blocking

Can a thread *block* when it invokes an interface in a function?
1. Can the thread block awaiting some more complicated event, potentially forever!
2. Block on a lock?
   So long as it is a lock with *progress guarantees*, we don't count this as "blocking".

---

## Blocking vs. Non-blocking

Blocking:
- *requires* multi-threaded client code if it wants to efficiently work with multiple (blocking) resources.
- Terse and simple if only working with a single resource.

Non-blocking:
- Returns immediately, even if operation cannot be performed yet.
- Degenerates into active polling.

What are the bad execution scenarios in the following?

---

```c [2-10|13-20|1-20]
/* What are the bad execution scenarios in each of the following? */

int
blocking(res_t r1, res_t r2)
{
	while (1) {
		blocking_operation(r1);
		blocking_operation(r2);
	}
}


int
nonblocking(res_t r1, res_t r2)
{
	while (1) {
		nonblocking_operation(r1);
		nonblocking_operation(r2);
	}
}
```

---

## Concurrency in Interfaces: Events

- Is a function potentially *blocking*?
- Does a thread block on a specific resource?
- What **events** trigger the unblocking/waking on a resource?

Generally:
- What **events** does a component define?
- What *triggers* those **events**?
- How does one *await* or block on a (or a set of) **event**(s)?

---

## Event Aggregation

Concurrent server (`nginx`):
- Want to await an event on *any* client's socket
- Can't block on a single client's socket

Primitives: `epoll`/`poll`/`select`
- await an event on *at least one* of a *set* of events
- return the *set of triggered events*

---

## KV Store + Events

We want to
1. associate an event with a resource (a `value`), and
2. trigger the event when there is an *update* on the value.

Challenge:
- `kv_put` *transparently* creates a value
- How do we know when/if we associate an event with it?
- Orthogonality$\to$ `kv_add` & `kv_update`

---

```c [1-4|6-7|31-38|45,57|70-81|89-91]
/* Client APIs */
evt_t evt_create(void *data);
void evt_delete(evt_t e);
evt_t *evt_await(evt_t grp, evt_prop_t p);

/* Server API */
int evt_trigger(evt_t e, evt_prop_t p);

/* Server component, extending the key-value store */
value_t
kv_get(key_t k)
{
	value_t v = VALUE_NULL;
	struct client_data *c = clients[client_id()];

	mutex_lock(&c->lock);
	val = ht_get(&c->ht, k);
	if (val) v = val->v;
	mutex_unlock(&c->lock);

	return v;
}

int
kv_add(key_t k, value_t v, evt_t e)
{
	struct client_data *c = clients[client_id()];
	struct val *val;

	mutex_lock(&c->lock);
	val = ht_get(&c->ht, k);
	if (val) {
		mutex_unlock(&c->lock);
		return -1;
	}
	val = malloc(sizeof(struct val));
	*val = (struct val) { .v = v, .e = e };
	ht_put(&c->ht, k, val);
	mutex_unlock(&c->lock);

	return 0;
}

int
kv_update(key_t k, value_t v)
{
	struct client_data *c = clients[client_id()];
	struct val *val;
	int ret = -1;

	mutex_lock(&c->lock);
	val = ht_get(&c->ht, k);
	if (val) {
		val->v = v;
		ret = 0;
		/* This is an update event on the value */
		evt_trigger(val->e, EVT_UPDATE);
	}
	mutex_unlock(&c->lock);

	return ret;
}

/* The client */
int
main(void)
{
	evt_t *e1, *e2;

	e1 = evt_create(NULL);
	e2 = evt_create(NULL);

	kv_add(1, 1, e1);
	kv_add(2, 2, e2);

	thread_create(triggering_fn);

	while (1) {
		evt_await(e1);
		printf("progress w/ %lx...", kv_get(1));
	}
}

void
triggering_fn(void)
{
	val_t to_the_moon = 0;

	while (1) {
		kv_update(2, to_the_moon++);
	}
}

```

Notes:
Problem here is that we won't be notified of the event on the *other* resource.

---


```c [4|71-92]
/* Client APIs */
evt_t evt_create(void *data);
void evt_delete(evt_t e);
int evt_group(evt_t grp, evt_t e);
evt_t *evt_await(evt_t grp, evt_prop_t p);

/* Server API */
int evt_trigger(evt_t e, evt_prop_t p);

/* Server component, extending the key-value store */
value_t
kv_get(key_t k)
{
	value_t v = VALUE_NULL;
	struct client_data *c = clients[client_id()];

	mutex_lock(&c->lock);
	val = ht_get(&c->ht, k);
	if (val) v = val->v;
	mutex_unlock(&c->lock);

	return v;
}

int
kv_add(key_t k, value_t v, evt_t e)
{
	struct client_data *c = clients[client_id()];
	struct val *val;

	mutex_lock(&c->lock);
	val = ht_get(&c->ht, k);
	if (val) {
		mutex_unlock(&c->lock);
		return -1;
	}
	val = malloc(sizeof(struct val));
	*val = (struct val) { .v = v, .e = e };
	ht_put(&c->ht, k, val);
	mutex_unlock(&c->lock);

	return 0;
}

int
kv_update(key_t k, value_t v)
{
	struct client_data *c = clients[client_id()];
	struct val *val;
	int ret = -1;

	mutex_lock(&c->lock);
	val = ht_get(&c->ht, k);
	if (val) {
		val->v = v;
		ret = 0;
		/* This is an update event on the value */
		evt_trigger(val->e, EVT_UPDATE);
	}
	mutex_unlock(&c->lock);

	return ret;
}

/* The client */
int
main(void)
{
	evt_t *e1, *e2, *eg;

	/*
	 * When the events trigger, we're having them
	 * return to us thekey associated with the event!
	 */
	e1 = evt_create((void *)1);
	e2 = evt_create((void *)2);
	eg = evt_create(NULL);

	kv_add(1, 1, e1);
	kv_add(2, 2, e2);

	/* the group event tracks the set of other events. */
	evt_group(eg, e1);
	evt_group(eg, e2);

	thread_create(triggering_fn);

	while (1) {
		/* return's the `void *` passed in `evt_create` */
		key_t k = (key_t)evt_await(eg);
		printf("progress w/ %lx...", kv_get(k));
	}
}

void
triggering_fn(void)
{
	val_t to_the_moon = 0;

	while (1) {
		kv_update(2, to_the_moon++);
	}
}



```

Notes:
Now we're waiting on the *group or set of events*, not for a specific resource/event.

---

## Event Interfaces

- *Event-triggered* - tell me when the resource status *changes* (i.e. a value is updated)
- *Level-triggered* - tell me all of the resources in a specific *state*

Level-triggered is *idempotent*, making it easier to write applications
- `select` returns the set of all `fd`s in specific states.

---

## Level- vs. Event-Triggered

Level-triggered (idempotent):
```python []
assert(select(evts) == select(evts))
```

Event-triggered:
```python []
assert(select(evts) != None)
assert(select(evts) = None)
```

Notes:
Event-triggered: once we read out an event, it won't be reported anymore!

Recall, when a function isn't idempotent, we're expecting the client to track state.

---

## Level-Triggered Events

```python []
evts = [e0, e1, ...]

# Event loop!
while True:
	e = select(evts).first()
	handle_event(e)
```

Notes:
If event triggered, we'd lose all events after the first for any given call to select

---

## Event-Triggered Events

```python []
evts = [e0, e1, ...]

# Event loop!
while True:
	s = select(evts)
	while s.len > 0:
		e = pop(s)
		handle_event(e)
```

Notes:
Have to create our own data-structure to track events we've been told about because *we won't be told about them again* if their state doesn't change.

---

## Orthogonality: Events & Timers

What if:
- we want to block awaiting events,
- or to wake after a span of time if no event happened?

```c []
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```

`timeout` enables this *timed block*!

-v-

See the [thorough discussion](https://gwu-cs-advos.github.io/sysprog/#revisiting-ipc-with-multiple-clients) on event management and `poll`.

---

## Orthogonality: Events & Timers

Downsides:
- Complexity: Now everyone who uses `epoll_wait` must worry about `timeout` (even if they don't need it)
- Not Orthogonal: We might want a complex pattern of timeouts, but an `int` doesn't get us there (periodic, real-time, exec_time, etc...)

All users of `epoll_wait` must use a half-baked timeout API.
- Timeout logic must *be baked into the event loop!!!*

---

## Orthogonality: Events & Timers

Alternatively, separate events and timers:
```c []
int timerfd_create(int clockid, int flags);
```
More useful way to set time (periodic, based on execution time, etc...).

Returns an `fd`, which we can add to the event fd!!!

---

```c [1-8|10-13|15-17|19-21|26-27|29-33|35]
tfd = timerfd_create(CLOCK_REALTIME, 0);
struct itimerspec  now;
clock_gettime(CLOCK_REALTIME, &now);
struct itimerspec  new_value = (struct itimerspec) {
	.it_value.tv_sec = now.tv_sec + atoi(argv[1]),
    .it_value.tv_nsec = now.tv_nsec,
};
timerfd_settime(tfd, TFD_TIMER_ABSTIME, &new_value, NULL);

struct epoll_event evt_data;
int event_fd = epoll_create(1);
evt_data.data.fd = tfd;
epoll_ctl(event_fd, EPOLL_CTL_ADD, tfd, &evt_data);

/* previously opened socket, e.g. on port 80 */
evt_data.data.fd = MAIN_SOCKET;
epoll_ctl(event_fd, EPOLL_CTL_ADD, MAIN_SOCKET, &evt_data);

while (1) {
	struct epoll_event evts[16];
	nevts = epoll_wait(event_fd, evts, 16, -1);
	for (i = 0; i < nevts; i++) {
		int efd = evts[i].data.fd;

		/* events returned in evts */
		if (efd == tfd) {
			handle_timer();
		} else if (efd == MAIN_SOCKET) {
			struct epoll_event socket_evt;

			fd = accept(MAIN_SOCKET);
			socket_evt.data.fd = fd;
			epoll_ctl(event_fd, EPOLL_CTL_ADD, fd, &socket_evt);
		} else {
			handle_client(efd);
		}
	}
}
```

---

## Event Management Summary

APIs
- Edge-triggered: notification on update
- Level-triggered: Ask for state of all fds

Composition
- Modern Linux enables fd-based access to most resources
  - memory, timers, processes, events, signals, ...
- Enables composition with event management

---

# Algebraic Properties

-v-

- Apache Beam requires some operators to be [commutative and associative](https://beam.apache.org/documentation/programming-guide/#combine)
- The [*scalable commutativity rule*](https://dl.acm.org/doi/10.1145/2699681) guides us to understand if an operation can be implemented scalably
- [Conflict-free replicated datatypes (CRDTs)](https://hydro.run/papers/keep-calm-and-crdt-on.pdf) enable many parallel operations on a core data-structure (multiple people modifying google docs at the same time), and guarantee eventual consistency. The “merge” operator is commutative, associative, and idempotent, and operations must be monotonic to be reasonably viewable.

---

## The Maths

Properties over *function* "$\circ$"
| Property | | Constraints |
|----------|--:| -------------|
| Commutativity | $\forall f, g,$ | $f \circ g = g \circ f$ |
| Associativity | $\forall i, j, k,$ | $(i \circ (j \circ k)) = ((i \circ j) \circ k)$ |
| Idempotency | $\forall h,$ | $ h \circ h = h$ |

- $f, g, h$ - system calls
- $i, j, k$ - system calls with inputs in a shared namespace

---

## Applied to System *Interfaces*

Definitions

- $\circ $ = system (a component) with some state
- $f$ = interface function (e.g. syscall) w/ specific set of arguments
- $f, g, h$ are separate *functions* (or functions with different arguments) in a system's interface

---

## Applied to System *Interfaces* II

Interpretation:

- $f \circ g \triangleq$ "System executes $f$ then $g$"
  - each of $f$ & $g$ return appropriate values, and
  - $f$'s execution is reflected in system state
  - **execution ordering**
- $f \circ (g \circ h) \triangleq$ "$g$ & $h$ apply changes to shared namespace, the output of which is combined with $f$'s changes"
  - **order of data combination**

---

## Applied to System *Interfaces* III

"Do system call `write` and `read` commute?"
- can they be reordered, and return the same values?

"Do system calls `pwrite` on **disjoint portions** of a file associate?"
- can these be executed in any order and the state combined identically?

"Do system calls `pwrite` on **any portion** of a file associate?"

---

## Commutativity

Are operations *order invariant* and can be implemented scalably?

<div class="multicolumn"><div>

| core 0        | core 1        |
|---------------|---------------|
| 1: $\textsf{f}(...) \to r_0$ | 1: $\textsf{f}(...) \to r_1$ |
| | |
|  | 1: $\textsf{f}(...) \to r_0$ |
| 2: $\textsf{f}(...) \to r_1$ | |
| | |
|1: $\textsf{f}(...) \to r_0$ | |
| | 2: $\textsf{f}(...) \to r_1$|

</div><div>

- System state is the same after each
- Operations aren't strictly related
- Any order of execution of operations is permissible

</div></div>

---

## Commutativity: Implications

**Scalability** -
An operation's logic doesn't impact another's.

- They modify no shared state - no locks!
- They should be able to execute independently - scalably.

---

## Commutivity Example

POSIX cannot be implemented scalably as `fd` allocation is not commutative
> A new `fd` should have the lowest available unused `fd` id.

So `dup` will always allocate into the lowest `fd`.
- Example from `xv6`

---

```c [3|5,6,12,13]
	case PIPE:
		pcmd = (struct pipecmd *)cmd;
		if (pipe(p) < 0) panic("pipe");
		if (fork1() == 0) {
			close(1);
			dup(p[1]);
			close(p[0]);
			close(p[1]);
			runcmd(pcmd->left);
		}
		if (fork1() == 0) {
			close(0);
			dup(p[0]);
			close(p[0]);
			close(p[1]);
			runcmd(pcmd->right);
		}
		close(p[0]);
		close(p[1]);
		wait(0);
		wait(0);
		break;
```

---

## `fd` Allocation not Commutative

Core 0: `fd0 = dup(...)`
Core 1: `fd1 = dup(...)`

Either can be true:
- `fd0 < fd1`
- `fd1 < fd0`

Must be a lock *serializing* the allocation!

---

## Modern Linux

1. `dup2` to choose the `fd`
2. Ignore POSIX!!!

---

## Associativity

Does the order in which we apply data updates of related operations impact the result?

No? $\to$ *Associative*

$f \circ g \circ h \circ i = (f \circ g) \circ (h \circ i)$
- parallel execution of $f \circ g$ and $h \circ i$

---

Examples:
- Key-value `get`s are associative:
  - ordering of operations only on put$\to$get `kv_put(k, ...); kv_get(k)`
- DB reads are associative, write$\to$read operations aren't
  - Replicated readers w/ single writer
- Stacks and queues are *not* associative:
  - order of push/pop/enqueue/dequeue changes output

Parallel/distributed KVs are common, not so much queue/stack

<!-- - $\textsf{block_read}(b); \textsf{block_read}(b'); \textsf{block_read}(b''), b \neq b' \neq b''$ -->
<!-- - Same for $\textsf{block_write}$ -->
<!-- - Same for $\textsf{block_read}$ & $\textsf{block_write}$ with any $b, b', b''$ -->
<!-- - $(\textsf{block_read}(b); \textsf{block_read}(b');) \textsf{block_write}(b) \neq$ -->
<!--   $\textsf{block_read}(b); (\textsf{block_read}(b'); \textsf{block_write}(b))$ -->

---

## Associativity: Implications

**Parallel execution**
- If the result is the same regardless how we "split up" execution, we can choose the execution order.
- Can enable *distributed execution* of operations

**Separate batch processing**
- Collections of operations can be executed together, then the output merged.
- This enables flexible *batch processing* of operations.

---

## Idempotency

Can operations be *replayed*?

Example:
- $\textsf{read}(fd, b, 1); \textsf{read}(fd, b, 1); \neq \textsf{read}(fd, b, 1, 0);$
- $\textsf{pread}(fd, b, 1, 0); \textsf{pread}(fd, b, 1, 0); = \textsf{pread}(fd, b, 1, 0);$
- $\textsf{select}(...); \textsf{select}(...); = \textsf{select}(...); $ <!-- .element: class="fragment" data-fragment-index="1" -->
- $\textsf{dup}(1); \textsf{dup}(1); \neq \textsf{dup}(1)$ <!-- .element: class="fragment" data-fragment-index="2" -->
- $\textsf{dup2}(1, 10); \textsf{dup2}(1, 10); \neq \textsf{dup2}(1, 10); $ <!-- .element: class="fragment" data-fragment-index="2" -->
- REST GET operations <!-- .element: class="fragment" data-fragment-index="3" -->
- SQL SELECT queries <!-- .element: class="fragment" data-fragment-index="3" -->

---

## Idempotency: System Implications

**Cacheability**

- No idempotency? $\to$ something being updated
- Nothing being updated? $\to$ can *cache* the output (rather than compute it)

**Burden of client tracking**

- Does the client need to track data, or can the server just be asked for it?
- Idempotent abstractions can lead to less state tracked in client ($\textsf{select}/\textsf{poll}$ vs. $\textsf{epoll}$)

---

# Interface/Implementation Security

---

## Security in Hierarchical Systems

Security:
- a hierarchically structured system
- "join-point" component that provides vertical isolation for "subsystems"

> *How might we describe if this component's interface/implementation provides "security" or not?*

<div class="center">

```mermaid
block-beta
	columns 2
	block p0["Not Classified"] end
	block p1["Classified"] end
	p3["Secure Component"]:2
```

</div>

---

## Confidentiality

Secure Component:
- $S_l$ and $S_h$ are subsystems with low and high security access
- Confidentiality: if we assume compromised computations in $S_l$ and $S_h$, $S_h$ should not be able to **pass information** to $S_l$

> If $S_l$ ever sees *varying results* depending on *varying actions* by $S_h$ , that can be used to send (at least) a bit of information from $S_h$ to $S_l$.

---

## Example

Don't want high-security (high-confidentiality) documents (e.g passwords) to end up in low-security, and out on the net!

---

## Non-Interference for Information Flow

- Non-Interference: policy encoding the previous policy
- Prevent the flow of information from $S_h$ to $S_l$

---

## Interface Formulation

$F^s_i(\hat{x}) \to a$

- $F^s_i$: interface functions in secure component
	- $s \in \{h, l\}$ = security level
- $\hat{x}$: inputs
- $a$: outputs
- $\circ$ - *commutative* execution in secure component

---

## Interface Formulation

$F^s_i(\hat{x}) \to a$

- $F^s_i$: interface functions in secure component
	- $s \in \{h, l\}$ = security level
- $\hat{x}$: inputs
- $a$: outputs
- $\circ$ - *commutative* execution in secure component

$\forall \hat{x}, \hat{y}, F^l_i, F^h_j, \exists a \hspace{1em} F^l_i(\hat{x}) \to a \circ F^h_j(\hat{y}) \to *$

---

$\forall \color{red}{\hat{x}, \hat{y}}, F^l_i, F^h_j, \exists a \hspace{1em} \color{red}{F^l_i(\hat{x})} \to a \circ \color{red}{F^h_j(\hat{y})} \to *$
- For all possible inputs to the interface functions

---

$\forall \color{red}{\hat{x}, \hat{y}}, F^l_i, F^h_j, \exists a \hspace{1em} \color{red}{F^l_i(\hat{x})} \to a \circ \color{red}{F^h_j(\hat{y})} \to *$
- For all possible inputs to the interface functions

$\forall \hat{x}, \hat{y}, F^l_i, F^h_j, \exists \color{red}{a} \hspace{1em} F^l_i(\hat{x}) \color{red}{\to a} \circ F^h_j(\hat{y}) \to *$
- The low security subsystem's output for a specific input $\hat{x}$

---

$\forall \color{red}{\hat{x}, \hat{y}}, F^l_i, F^h_j, \exists a \hspace{1em} \color{red}{F^l_i(\hat{x})} \to a \circ \color{red}{F^h_j(\hat{y})} \to *$
- For all possible inputs to the interface functions

$\forall \hat{x}, \hat{y}, F^l_i, F^h_j, \exists \color{red}{a} \hspace{1em} F^l_i(\hat{x}) \color{red}{\to a} \circ F^h_j(\hat{y}) \to *$
- The low security subsystem's output for a specific input $\hat{x}$

$\forall \hat{x}, \hat{y}, F^l_i, F^h_j, \exists a \hspace{1em} F^l_i(\hat{x}) \to a \color{red}{\circ} F^h_j(\hat{y}) \to *$
- Must not change regardless order of function execution (commutativity)

---

## Non-Interference Restricts Information Flow

- No action in high or low security domains...
- can cause information to pass...
- from high-security to low-security!

---

## Implementations: Separation Kernel

- Separate namespaces, ...
- to disjoint sets of resources.

$\to$ no shared resources/namespaces, non-interference!

---

## Implementations: Add Communication

Separation kernel is limited
- No information flow in *either direction*

Communication facilities
- Pipe from $S_l \to S_h$
  - Not from $S_h \to S_l$!
- ...not sufficient, why?

---

## Nuanced Communication

Pipe from $S_l \to S_h$

- never blocks $S_l$
- doesn't return to $S_l$ if data was queued or not

Non-interference can be hard to get right!
- Formal verification as in *Nickel*

---

# Implementation Optimizations

---

## Data Aggregation: Interface Usage Pattern

```c []
data_update(handle, data);
data_update(handle, data);
data_update(handle, data);
data_update(handle, data);
/* ... */
data_update(handle, data);
```

Might include "read" operations, and/or mixes of operations

---

## Data Aggregation: Batching

$t$ = interface cost, $c$ = communication cost

Example: `syscall` $t = 200cyc$, `swtch` $c = 1000cyc$

Cost:
- send $N$ messages, one at a time: $N \times (t + c)$
- batch $N$ messages: $Nt + c$

Latency for first message:
- send $N$ messages one at a time: $t + c$
- send $N$ messages: $Nt + c$

Batching trades latency for throughput <!-- .element: class="fragment" data-fragment-index="1" -->

---

## **Data** Aggregation: Batching Policies

- Batch based on concurrency up to $N$ - pipes
- Time-bounded batching - up to $N$ or $t$

---

## Data Aggregation: Temporal Locality

Think:
- Cache working set for "client" code: $N$ lines
- Cache working set for "server" code: $M$ lines

No batching: Each operation requires $N + M$ lines

Batching: Number of operations use $N$ lines, **then** a number of operations use $M$ lines

Biggest performance benefit (cache size = $L$):
- $L > N \wedge L > M \wedge L < N + M$

---

## Data Aggregation: Examples

- Buffered I/O - batch up to `\n` (`fwrite`, `printf`)
- DMA ring buffers - batch up to a fixed # of I/O req/resp
- UNIX pipes - batch up till pipe buffer size

---

## Example: No Batching

```mermaid
sequenceDiagram
    autonumber
    P0->>kernel: write(...)
    kernel->>P1: swtch(P1)
    P1->>kernel: read(...)
    P1->>kernel: write(...)
    kernel->>P0: swtch(P0)
    P0->>kernel: read(...)
```

---

## Example: Batching

```mermaid
sequenceDiagram
	autonumber
    P0->>kernel: write(...)
	P0->>kernel: write(...)
    kernel->>P1: swtch(P1)
	P1->>kernel: read(...) + write(...)
	P1->>kernel: read(...) + write(...)
	kernel->>P0: swtch(P0)
	P0->>kernel: read(...)
	P0->>kernel: read(...)
```

---

## Data Aggregation: Optimization

Associative interfaces
- can reorder batched data processing
- use multiple cores effectively

Idempotent interfaces
- "batch logic" can omit redundant operations in a batch

---

## Action Aggregation: Interface Usage Pattern

```c []
f(handle, a)
g(handle, b)
/* ...or... */
r = h(handle, a)
i(handle, r)
```

---

## **Action** Aggregation

*Optimization*:
- Find common sequences of operations, and
- move their **logic** into server components.

No aggregation of $N$ interface calls (each $t$):
- $Nt$ overhead

Aggregation of $N$ calls into one:
- $t$ overhead

---

## Action Aggregation: Interface Usage Pattern

```c []
f(handle, a)
g(handle, b)
/* ...or... */
r = h(handle, a)
i(handle, r)
```

becomes

```c []
f_and_g(handle, a, b)
h_and_i(handle, a)
```

---

## Action Aggregation: Examples

- Stored procedures in DBs
- `ebpf` programs in the Linux kernel
- graphql to specify complex, nested queries
- [Zygote](https://source.android.com/docs/core/runtime/zygote) processes in Android
  - aggregated initialization actions
  - to quickly create pre-initialized app startup
- Microkernel IPC
  - `send` + `recv` (like `read` & `write`) versus
  - `call` and `reply_and_wait`

---

## Microkernel IPC (L4-style)

```mermaid
sequenceDiagram
	autonumber
	c->>kernel: send(msg)
	kernel-->c: ret = send(msg)
	c->>kernel: recv(...)
	Note over kernel: swtch(s)
	kernel-->s: recv(msg)
	s->>kernel: send(msg')
	Note over kernel: swtch(c)
	kernel-->c: ret = recv(msg')
```

---

## Microkernel IPC (L4-style)

```mermaid
sequenceDiagram
	autonumber
	c->>kernel: call(msg)
	Note over kernel: swtch(s)
	kernel-->s: msg = reply_and_wait(...)
	s->>kernel: reply_and_wait(msg')
	Note over kernel: swtch(c)
	kernel-->c: msg' = call(msg)
```

---

## Action Aggregation: Perspective

**Optimization**:
- Take orthogonal functions or APIs, and
- Add *non-orthogonal APIs* that combine *performance-sensitive* operations

User-definition of fastpaths
- SQL stored procedures, `ebpf`, graphql

Component-definition of fastpaths
- microkernel `call`/`reply_and_wait`
- zygote

---

# Summary: Interfaces & Component Design

---

```mermaid
mindmap
	root((Interfaces))
		virtualization
		liveness/data placement
			borrowing
			ownership
			copying
		concurrency
			blocking
			per-client
			events
				event
				level
	    algebraic properties
			commutativity
			associativity
			idempotency
		aggregation
			data
				batching
				latency vs. throughput
			fastpath actions
				user-defined
				component-defined

```

-v-

## Case Study: Scheduling

Four systems that handle scheduling differently:

1. Linux
2. Hydra
3. Exokernel
4. Composite microkernel

Questions
- What are the trade-offs for each?
- What is difficult or impossible?
- What is unclear in the design?

---

1. *Linux* - fixed set of policies (general-purpose, EDF, fixed priority), interfaces to set per-thread parameters (priority, etc...)
   - `cgroups` can assign rate-limits to collections of threads, either weighted-fair, or a limit of N units over M
2. *Hydra Scheduling* - a scheduling policy process can start/stop their processes, and alter their parameters
   - Kernel mechanism selects the policy process, and schedules its threads according to their parameters
   - parameters include timeslice and priority
   - execute until timeslice expires, or process blocks
   - kernel has fairness policies between policy processes, each w/ N out of M time units

---

3. *Exokernel Scheduling* - The kernel maintains a timeline of timeslices, processes are given a proportion of time, and they address which *slots* in the timeline in the future they want to use.
   - The kernel simply switches to the corresponding process at each timeslice.
4. *Composite Microkernel* - A scheduler component has the ability to dispatch directly to its threads.
   - When a thread wants to block, it uses IPC to invoke the scheduler, which can dispatch to another thread.
   - The scheduler, when dispatching can program the timer interrupt to fire at $c$ cycles in the future.
   - There can be multiple schedulers.
