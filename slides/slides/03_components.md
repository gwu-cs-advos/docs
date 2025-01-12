---
title: Organizing Components
---

# Modularity: Organizing Components

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2024, All rights reserved

</div>

---

## Systems Must Have Multiple Components

- What is the granularity of components?
- How are component's related?

---

## How Should we Organize Software Components?

---

## Complexity with Components

Well-designed, deep components constrain complexity for their abstractions
- But the *relationships* between components can be complex
- $N$ components that can talk to any other would need to track $N^2$ interactions
- Broad component *coupling* has [security](https://www.hbs.edu/faculty/Publication%20Files/18-031_78569c7d-fc0c-4e1e-bd79-143bc507112a.pdf) [implications](https://link.springer.com/chapter/10.1007/978-3-319-62105-0_4) (e.g. in Chrome)

---

## :one: Layering

<div class="multicolumn">
<div>

Total order of components:
- System has $L$ layers
- Component $c_i$ is at layer $l_i < L$
- $c_i$ *only* invokes functions in $c_j$ where $l_j = l_i - 1$

</div>
<div>

```mermaid
block-beta
	columns 1
	block:L2
		c0(["C"])
	end
	space
	block:L1
		c1(["C"])
	end
	space
	block:L0
		c2(["C"])
	end
	c0 --> L1
	c1 --> L0
```

</div></div>

---

## Layering

A $c_i$ with $l_i$
- Provides interface used by $c_k$ where $l_k = l_i + 1$
- Impacted by logic/failures in $c_j$ where $l_j < l_i$

Constrain complexity:
- Each layer considers abstractions only at next layer
- Each layer abstracts over abstractions in next layer

-v-

- ["The structure of the “THE”-multiprogramming system"](https://dl.acm.org/doi/10.1145/800001.811672) by Dijkstra

---

## Layering Generalization

<div class="multicolumn">
<div>

$c_i$ *only* invokes functions in $c_j$ where

$$
l_j = l_i - 1 \vee l_j = l_i
$$

Trade-offs?

</div>
<div>

```mermaid
block-beta
	columns 1
	block:L2
		c0(["C"])
		c1(["C"])
		c2(["C"])
	end
	space
	block:L1
		c3(["C"])
		c4(["C"])
	end
	space
	block:L0
		c5(["C"])
		c6(["C"])
		c7(["C"])
	end
	c0 --> L1
	c1 --> L1
	c2 --> L1
	c3 --> L0
	c4 --> L0
```

</div></div>

---

## Layering + Passthrough

<div class="multicolumn">
<div>

Allow invocation of components in *any lower level*:
- $c_i$ *only* invokes functions in $c_j$ where $l_j \leq l_i$

Trade-offs?

</div>
<div>

```mermaid
block-beta
	columns 1
	block:L2
		c0(["C"])
		c1(["C"])
		c2(["C"])
	end
	space
	block:L1
		c3(["C"])
		c4(["C"])
	end
	space
	block:L0
		c5(["C"])
		c6(["C"])
		c7(["C"])
	end
	c0 --> L1
	c0 --> L0
	c1 --> L1
	c1 --> L0
	c2 --> L1
	c3 --> L0
	c4 --> L0
```

</div></div>

---

## Layering Examples???


---

## Layering Examples

<div class="multicolumn">
<div>

Relatively common
- *Complexity*: Each layer implemented for the abstractions at lower-layers
- *Faults*: Each layer must "trust" the layers below to work correctly

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

</div>
</div>

---

## Deep Components w/ Layers

<div class="multicolumn">
<div>

Layering within monoliths
- Manage complexity
- Abstraction tower
- Discipline through code-review

</div>
<div>

```mermaid
block-beta
	block:VMKern
		columns 1
		sc["syscall demux"]
		fd["fd lookup"]
		vfs["VFS interface"]
		fs["File System"]
		cache["Block Cache"]
		driver["SSD driver"]
	end
```

</div>
</div>

---

## Deep Components w/ Layers

<div class="multicolumn">
<div>

Layering within monoliths
- Manage complexity
- Abstraction tower
- "Tech stack"

</div>
<div>

```mermaid
block-beta
	block:Server
		columns 1
		ws["\nNginx\n\n"]
		flask["\nPython Flask\n\n"]
		db["\nPostgres\n\n"]
	end
```

</div>
</div>

---

## Layering Trade-offs

:+1: Strong organization of components, constrain complexity

:-1: Downsides
- Multiple applications might require different abstractions
- Little isolation between applications/users
- Unrelated abstractions don't fit into levels
  - Which should be in higher/lower layers: Networking or Filesystem

:question: What other options do we have?

---

## :two: Hierarchy

<div class="center">

```mermaid
block-beta
	block:S
		columns 2
		block:U0 p0["app"] end
		block:U1 p1["app"] end
		k0["kernel"]:2
	end
```

</div>

---

## Hierarchy

<div class="center">

```mermaid
block-beta
	columns 3
	block p0["app"] end
	block p1["app"] end
	block p2["app"] end
	block:KERN:3
		columns 3
		space:1
		sc["syscall demux"]
		space:2
		fd["fd lookup"]
		space:2
		vfs["VFS interface"]
		space:1

	    pipe["Pipes"]
		fs["File System"]
		pfs["Pseudo-File System"]
		space:1

		cache["Block Cache"]
		space:2

		driver["SSD driver"]
	end
```

</div>

---

## Hierarchy

<div class="multicolumn">
<div>

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

</div><div>

What if we had a VMM per VM?
- What properties of the system does that change?

</div></div>

---

## Hierarchy

<div class="multicolumn">
<div>

Tree-like structure system around per-principal layers enabling principal-centric abstraction.

Often paired with *isolation* facilities
- $c_i$ can only communicate with its associated layers

</div><div>

```mermaid
flowchart TB
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

## Hierarchy - Specialization

<div class="multicolumn">
<div>

Principal's abstractions customized and principal-specific
- Linux VM vs. Windows VM
- Qemu vs. Firecracker

Your app w/ your libs in docker in VM in VMM in host

</div><div>

```mermaid
flowchart TB
	g["g :white_check_mark:"]
	h["h :white_check_mark:"]
	i["i :white_check_mark:"]
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

-v-

- *Principles of Computer System Design*, Saltzer & Kaashoek
- *Hints and Principles for Computer System Design*, Lampson

---

##  Hierarchy - Trade-offs

<div class="multicolumn">
<div>

:one: vs :two: vs :three:
- Abstraction properties?
- Isolation properties?

</div><div>

```mermaid
flowchart TB
	a["a :two:"]
	e["e :two:"]
	i["i :one: :two: :three:"]
	k["k :three:"]
	c["c :one:"]
	f["f :one:"]
	h["h :one:"]
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

Notes:
- :one: `c` is within a more abstract environment, provided by layers below it, and, all things being equal, will be more isolated given the more abstracted access to resources.
- :three: `k` has more direct access to system resources as the abstractions it accesses are more shallow. This will enable less of a semantic gap, but might have an impact on isolation properties.
- :two: somewhere in-between.

---

## Hierarchy - Trade-offs

<div class="multicolumn">
<div>

:one: vs :two: vs :three:
- Abstraction properties?
- Isolation properties?

</div><div>

```mermaid
flowchart TB
	a["DHCP\nserver :two:"]
	b["VoIP\nserver"]
	e["IP + UDP :two:"]
	i["NIC Device Driver :one: :two: :three:"]
	k["Router :three:"]
	c["HTTP\nserver :one:"]
	d["ssh\nserver"]
	f["TCP :one:"]
	h["IP + UDP :one:"]
	g["Streaming\nmedia"]
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

## Hierarchy Downsides

- Challenging interactions between different tree branches
  - *deduplication* avoids memory copies
  - fast inter-VM communication
- What "layer" of hierarchy is right for an app?
  - Router: relatively direct access to NIC
  - HTTP server: desires significant
- Components with multiple "children" must provide resource isolation
- Same layering downsides - Unrelated abstractions don't fit into levels (net vs. FS)

---

## Hierarchy Trade-offs

| Layers | Hierarchy | Issue |
|--------------------|--------------------|---------------------------------------------------|
| :white_check_mark: | :white_check_mark: | Constrain complexity       |
| :white_check_mark: | :warning:           | Easy resource sharing (across subtrees)           |
| :white_check_mark: | :warning:           | No resource isolation       |
| :x:           | :white_check_mark: | Application (subtree) isolation         |
| :x:           | :white_check_mark: | Appllication (subtree) abstraction specialization |
| :x:           | :x:           | Avoid strict ordering of abstractions             |

---

# System as DAG of Components

---

## Components

<div class="multicolumn">
<div>

**Component**
- *Interface(s)*
- *Implementation*
- *Dependencies* on other Interfaces

</div><div>

```mermaid
flowchart TB

A[\"Interface A"/]
B[\"Interface B"/]
C[\"Interface C"/]
D[/"Interface D"\]
E[/"Interface E"\]
F(["\nComponent\nImplementation\n\n"])

A -.-> F
B -.-> F
C -.-> F
F -.-> D
F -.-> E
```

</div></div>

---

## Component Dependencies

<div class="center">

```mermaid
flowchart TB

A[\"Interface A"/]
D[/"Interface D"\]
F(["\nComponent 0\nImplementation\n\n"])
E[\"Interface D"/]
G(["\nComponent 1\nImplementation\n\n"])

H(["\nComponent 0\nImplementation\n\n"])
I(["\nComponent 1\nImplementation\n\n"])


A -.-> F
F -.-> D
D --> E
E -.-> G

H --> I
```

</div>

---

## System as Components

<div class="multicolumn">
<div>

```mermaid
flowchart TB

H(["\nNet\nstack\n\n"])
I(["\nNIC device\ndriver\n\n"])
A(["\nFilesystem\n\n"])
B(["\nSSD\ndriver\n\n"])
C(["\nHTTP\nwebserver\n\n"])
D(["\nRouting\n\n"])
E(["\nShared\nmemory\n\n"])

H --> I
C-->H
C-->A
A-->B
D-->I
D-->E
C-->E
```

</div><div>

- Dependencies based on explicit needs
- Explicit sharing (e.g. shared memory)
- No artificial layering (DAG)

</div></div>

---

<img src="resources/01_lkm.png">

---

## Software Design vs. System Structure

Managing complexity is a software goal
- How do you organize your program?
- Component $\approx$ Object

...but systems require management of *isolation*.
- How should we *structure* the system?

---

## System Structure

Where are *isolation boundaries* between components?

Where should controlled *communication* be allowed?

Where should *resource sharing* be allowed?

---

# Summary: Structure

---

```mermaid
mindmap
root ((system structure))
	layering
		total order structure
		pass-through
		properties
			limit complexity
			total ordering
			fixed abstractions
		mechanisms
			dual-mode
			syscalls/exceptions
			VM HW
	hierarchy
		tree-based structure
		properties
			per-application specialization
			required total ordering
		mechanisms
			page-tables
			ASIDs
	components
		DAG-based structure
		interfaces
		properties
			specialization
			minimality
```