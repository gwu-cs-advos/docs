---
title: Retrospective
---

# Class Retrospective

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2025, All rights reserved

</div>

---

## Topics

- System Complexity
- Abstraction
- Modularity & organizing components
- System structure & isolation
- Interfaces

---

## System Complexity

- Massive systems, how do we manage them?
- Complexity causes - mass, quadratic state/operation management
  - Essential/Intrinsic vs. Accidental
  - Technical debt
- Complexity in system, or in client?
- Semantic gap
- Leaky abstractions

> But how do we handle complexity?

---

## Abstraction

- Interface, implementation, & resource specifications
- Shallow vs. deep components
  - wide interface & shallow - not much abstraction, potentially *leaky*
  - deep component - implementation does heavy lifting, potential *semantic gap*
  - deep components w/ leaky additions (`ioctl`)

> But how do we combine abstractions?

---

## Modularity: Organizing Components

- *Abstraction*: contain complexity into component
- *Component organization*: how are component's related?
  - layering (passthrough) - simplicity & fault containment
  - hierarchy - specialization & isolation, challenges sharing & specialization flexibility
  - directed-acyclic graph - interface-driven linkage, specialization,
	- avoid inappropriate leaky abstractions & semantic gaps

> But how do we isolate components?

---

## System Structure & Isolation

- How design & implement isolation boundaries between components?
- Resource Isolation
  - memory, control flow, time, abstract resources
- Trust
  - understanding implications of component failure
  - viral - spreads where there is no isolation & transitively
- Vertical isolation - core of layered isolation, separate "manager" from "client"
- Horizontal isolation - further contain faults between "clients"

---

## Isolation Requirements/Properties

- Resource naming - pointers, strings, descriptors
- Resource binding - name $\leftrightarrow$ resource association, resolution ($f(n) \to r)$)
  - implemented in manager component
- Shared namespaces - sharing, interaction, fault spreading, ambient authority
- Partitioned/separate namespaces - capabilities & fine-grained access control, delegation
- Client authentication - trustworthy client identity

> But how do we implement our namespaces and logic?

---

## Interface Properties

- Virtualization - abstraction abstraction
- Liveness & data placement (allocation)
- Concurrency
  - blocking/non-blocking
  - event management (level- vs edge-triggered)
- Algebraic properties
  - commutativity - scalable parallel execution
  - associativity - parallel processing of data
  - idempotency - caching & client simplicity

---

## Interface Properties II

- Security - non-interference
  - information flow
  - separation kernel
- Optimizations
  - data aggregation - batching
  - action aggregation - IPC, ebpf, etc...

---

## Interface & System Design

Powerful interfaces provide:

- Polymorphism - broadly useful to clients
  - specializable, general (deep) interface
- Composition - client flexibility in interface use
  - shared namespaces enable composing complex functionality from simple pieces
  - everything is a file, or file descriptor
- Separation of concerns/Orthongonality
- Separation of mechanism and policy
  - reach down to next "mechanism layer" to avoid semantic gaps, avoid adding leaks given this capability

---

## Advanced OS

- :flexed_biceps: read a ton of code
- :question: related the code to the class contents
- :crossed_fingers: better understand the design, implementation, and properties of *complex software systems*
  - :ninja: not just OSes
- :unlock: got confidence you can dive into any code base
  - :exploding_head: there's no magic here

<div class="center">

## :tada: Thank you!!! :tada: <!-- .element: class="fragment" data-fragment-index="1" -->

</div>
