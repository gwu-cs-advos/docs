---
title: Nickel and Komodo
---

# Nickel and Komodo

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2025, All rights reserved

</div>

---

## Nickel Review

Very simple...to a fault

---

## Alternative PID allocation policy

- `spawn` includes `lower_pid` and `high_pid` arguments
- "moves" that range of pids to be available to be used by that child
- `spawn` also includes the child pid to use!

Does this work? Does it avoid passing information?

Trade-offs?

---

# Komodo & Enclaves

---

## Enclaves Motivation

Hierarchical systems:
- Child components always *trust* parents
- Memory often comes from parents, execution, etc...
- Parents can likely access any of a child's memory

---

## Enclaves Motivation

Component-based systems:
- clients components always *trust* servers...
- ...*at least for the services the server provides*.

Examples:
- Server might provide memory? trust it to not corrupt client's memory
- Server might schedule? trust it to not starve client

---

## Enclaves

What if we want the client/child to not have to trust the server/parent?

Cloud systems:
- Client wants to run system, but doesn't want to trust the host.
- Host doesn't want to be subpoenaed to provide client data.

---

## Enclaves

But the client wants the server/parent/OS to set up its memory, and might want to use its I/O...

We want to use the OS, but not trust it!

---

## Komodo

- Komodo: An implementation of enclaves mostly in software
- Intel SGX is hardware that is used in the cloud

---

## Enclaves: I/O

Use the OSes I/O
- but protect confidentiality by encrypting all traffic
- not really a huge design point of Komodo

---

## Enclaves: Creation

The OS must create the enclave's computation
- the initial execution image
- How does the enclave know that the OS did so correctly?

*Attestation* is the act of validating that an image is what you think it is
- Think: hash the image, and make sure it matches the desired hash value
- Komodo-provided attestation

---

## Enclaves: Prevent OS from Accessing Enclave Memory

1. Confidentiality: it can!
   But the hardware will encrypt all memory as it is executed in the enclave (Intel SGX).
2. Komodo: Let the OS create the computation, but then "finalize" the enclave, and disallow OS access to it from that point on.

---

## Komodo Communication

Communication between OS and enclave
- shared memory

---

## Komodo Implementation: ARM Trustzone

Use Trustzone:
- Hardware isolation between "secure world" and "normal world".
- Each page is labeled as one or the other.
- "Mode transition instructions to switch from normal to secure.

Almost like another "mode" below kernel mode.

---

## Komodo

Run enclave in secure world, in user-level
- OS API based on retyping
- move pages out of normal into secure to create the enclave
