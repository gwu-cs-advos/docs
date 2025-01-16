\newpage

<!--
Copyright (c) 2021 by Gabe Parmer.

Redistribution of this file is permitted under the GNU General Public
License v2.
-->

# Operating Systems Security

We've discussed system abstraction and design.
We've identified the separation of software into modules, and orchestrating the coordination and isolation between modules as fundamental in providing inter-client isolation in spite of having shared services that can manage system resources.
We've also discussed capability based systems as a means to provide access control on primitive system resources, and Plan 9 namespaces as a means to control which resources (and which virtual resources provided by user-level 9P services) each process has access to.

However, we haven't thoroughly discussed how to think about *system security*.
This chapter will focus on this essential topic.

## References

- A foundational taxonomy and overview of security concepts: [The Protection of Information in Computer Systems](http://web.mit.edu/Saltzer/www/publications/protection/).
- Fantastic [OS security](https://pdfs.semanticscholar.org/3126/a5318ec8b478da08944b97358c7e3491c08d.pdf?_ga=2.180702311.1928750056.1609337029-963934370.1607645716) textbook by Trent Jaeger.
- Great OS textbook *overview* of security: an [intro](https://pages.cs.wisc.edu/~remzi/OSTEP/security-intro.pdf), [authentication](https://pages.cs.wisc.edu/~remzi/OSTEP/security-authentication.pdf), [access control](https://pages.cs.wisc.edu/~remzi/OSTEP/security-access.pdf), [crypto](https://pages.cs.wisc.edu/~remzi/OSTEP/security-crypto.pdf), and [distributed](https://pages.cs.wisc.edu/~remzi/OSTEP/security-distributed.pdf).
- A decent [overview](http://dance.csc.ncsu.edu/papers/CSUR2016.pdf) of more modern OS security techniques.
- Specific systems:

	- [Protection and the control of information sharing in multics](https://dl.acm.org/doi/10.1145/361011.361067)
	- [Flask](https://www.cs.cmu.edu/~dga/papers/flask-usenixsec99.pdf) which is the foundation for seLinux
	- [Asbestos](http://www.scs.stanford.edu/~dm/home/papers/efstathopoulos:asbestos.pdf)

## Concepts

System security revolves around defining and constraining which principals can access which data and resources.
We often intuitively think about cryptography when we discuss security.
Cryptography is often used to algorithmically limit which principals can access encrypted data.
We think about OS security as focusing around abstractions to provide isolation, and limit access to system resources.
We've already seen how *namespaces* are integral in this.
Namespaces prevent resources from even being addressed by a process when they aren't present.
*Capabilities* in a CBOS mediate access to primitive resources in the system (pages and other kernel resources).
*Hierarchical namespaces* define the set of file-like resources accessible to a process in Plan 9, and, importantly, abstract resources can be defined in user-level services, thus the system can be arbitrarily extended.
These are the *mechanisms* by which we provide isolation and prevent unconstrained resource accesses.

The next important question is how we determine *which resources are available in a principal's namespace*.
In Plan 9, a parent process that `rfork`s a child has the ability to use `unmount` to remove portions of the namespace (i.e. remove access to resources), and `bind` to craft the namespace.
We also have ways in which this access is organized and distributed using `exportfs` which chooses which subtree of the namespace it is serving up to an `import`.
In a CBOS, delegation and revocation define a set of mechanisms for resource namespace modification, and components use those mechanisms to enable access from other components to a subset of their resources.
In Composite, the capability manager can define its own policies for how resources can made accessible across components.

### CIA

A good "first cut" at what we're trying to accomplish in secure systems is that we're trying to enforce CIA:

- **Confidentiality.**
	This is the act of ensuring that only principals that should be able to see some data, can actually *read* the data.
	It is easy to see why encryption is often an important part of providing confidentiality, especially when communication crosses a network.
	It is important to realize that confidentiality is also one of the primary consideration when removing read access to files, or when removing files from a principal's namespace.
	When encryption keys or your password are held in a processes, we rely on memory isolation to ensure confidentiality.
- **Integrity.**
	Integrity's goal is to prevent principals from potentially corrupting information or data-structures that they should not have access to.
	Module protection domains also prevent *writes* to sensitive data from other principals, thus ensuring integrity.
	Integrity is also protected for files when principals are *not* given write access to them.
	Note that leaking out passwords, encryption keys, or contact lists are not threats to integrity.
- **Availability.**
	The ability of the system to provide service is its availability.
	If a principal is able to manipulate the system such that it cannot provide service (or takes longer to provide service than is useful) for other principals, then the system fails to provide availability.
	A Denial of Service (DoS) attack that continuously crashes a web-server, or triggers some execution path that takes too long and prevents servicing other clients, are both attacks on availability.

### Reference Monitor

The core of OS security is the **reference monitor** (*refmon*).
When a principal requests access to a resource, the *refmon* validates that it has the *privilege* to access the resource.
A *refmon* should have the following properties:

- *Complete Mediation* -
	All requests to access resources in the system must be checked, and allowed or denied based on the *refmon*'s mechanisms and policies.

	A CBOS's kernel is mainly focused on being the system's *refmon* (especially in Composite where scheduling is moved to user-level).
	Whenever a capability is activated, the kernel determines 1. if that capability exists and references a resource, and 2. how the resource can be operated on.
	The hardware plays a necessary part of the *refmon* as well.
	For example, each load and store is translated via the page-tables to the physical memory resource, which means that the hardware page-table walker, the TLB, and the interaction between the kernel and those hardware facilities are all part of the *refmon*.

	- The complexities in *refmon* implementation are many.
		Beyond traditional software correctness, one needs to worry about Time of Check to Time of Use ([TOCTTOU](http://www.watson.org/~robert/2007woot/)) bugs that open up small windows of *refmon* incorrectness.
		An amazing example: [dirtyCOW](https://dirtycow.ninja/).
	- Linux Security Modules ([LSM](http://www.cse.psu.edu/~trj1/cse544-s10/papers/lsm.pdf) and [its](https://www.kernel.org/doc/html/latest/security/lsm.html) [API](https://www.kernel.org/doc/html/latest/security/lsm-development.html)) try to simplify the reasoning behind the *refmon* in Linux.

- *Tamperproof* -
	The resource access decisions made by the *refmon* must be determined solely by the *refmon*'s own state, and its intended logic.
	Principals, and principal's actions must not be able to corrupt the decision making process of the *refmon*.
- *Verifiable/Trustworthy* -
	As the *refmon* is the core decision procedure for all resource access, we *must trust it*.
	Independent if a principal can tamper with it, it might have *bugs* which cause it to crash or make incorrect resource access decisions.
	Thus, we must take steps to ensure that it is maximally trustworthy.
	The concept of "trustworthiness" is hard to put your finger on, but it is clear that you trust code to perform its required logic if it is *formally verified*.
	For software that has been formally verified, the software's code is mathematically proven to correspond to a *specification* of the software's desired functionality^[Technically, we aim to prove that the specification is a refinement of the code.].
	There are other characteristics of software that might make it more trustworthy, but verification is one of the strongest ways.

In the end, of the *refmon* is compromised in any way (i.e. its decisions are coerced by a principal), then security at higher-levels in the system is generally not possible.
Its mechanisms and policies are the foundations for providing CIA in the system.

### Principles

To generate a secure system we typically follow a number of principles.
These are the modus operandi that we should follow by default, and depart from only with great care.

- **Complete Mediation.**
	As discussed above, we must ensure that all resource accesses must be properly authorized by the *refmon*.
	If the *refmon* is "out of the loop" of some resource access decisions, it can be challenging to maintain strong CIA.
	Note, that this implies that there are foolproof means for the *refmon* to know which principal is making the request, and to identify the requested resource.
	For example, a CBOS kernel must know which protection domain is requesting capability access, look up the access in the proper, associated capability table, and indexing into it to find the resource.
- **Least Privilege.**
	Each protection domain should have access to the minimal set of resources required to complete its task.
	There's no reason that a spreadsheet application should have access to the network, or a webserver to the ability to install software.
	Most processes in Linux should not have access to all of the almost 400 system calls (nor the almost 2000 in [Windows](https://github.com/j00ru/windows-syscalls)).
	Further, most processes should not have access to all of a user's files!
	This principle is important 1. to minimize the scope of integrity and confidentiality violations if a program is compromised by an attacker, and 2. to minimize the processes that must be inspected if a resource is found to be corrupted (this is related to the concept of "audits").
- **Economy of Mechanism** or "Keep it Simple Stupid" (KISS).
	Keep the design and implementation as simple and small as possible.
	There is a correspondence between system complexity, and the incidence of bugs.
	There seems to be a correspondence between system complexity (and lines of code), and effort to formally verify it (seL4 is 10K LoC in C, and 200K LoC in Isabelle/Hol proof language).
	The more edge-cases a design has, the harder it is to test, and the easier it is to gain false confidence in it -- "it works in all of the situations I tested!".
	The fewer bugs an implementation has, the less easily it will be compromised, and the easier it will be to fix.
- **Least Common Mechanism.**
	Minimize the number of services that are shared between two principals.
	Where a service is leveraged by multiple principals, it represents a potential means for one principal to negatively impact another.
	One might leverage a bug in the shared service to corrupt the other's data (integrity), to access its data (confidentiality), or to send the service into an infinite loop while it is servicing the other principal (availability).
	Each shared service requires significant scrutiny.
	Thus, such shared services should be minimized where possible.

	Note that this goal is often in opposition to efficient resource usage!
	If we follow the least common mechanism, we might have two components that manage separate pools of memory for two principals.
	What if one of the principals needs more memory than is in its pool, and the other is under-utilized?
	In contrast, having a single memory management component with a single pool that it can distribute among principals enables efficient memory allocations in response to dynamic demands, but is a single server whose compromise can impact all principals.
- **Separation of Privilege.**
	If the privilege to accomplish some task can be split up among different isolated protection domains such that they can coordinate to accomplish the task, system security will be increased.
	If one of the protection domains is compromised, an attacker will not have unfettered access to resources, and will still have to coordinate through well-defined APIs with the other protection domain.
	This will limit the extent to which it can access and manipulate resources.
	A system's file system can be isolated from the SSD device driver (in separate components) such that a fault in the driver will not corrupt all "in flight" I/O requests stored in the FS.
- **Minimal Trusted Computing Base (TCB).**
	The TCB is the set of all software and hardware that a given computation requires to be correct to accomplish its goals.
	In Linux, this includes all programs run as the same user as the current process, all those running as `root`, all of the `systemd` services whose function is leveraged by the process (or by libraries and programs it leverages), the kernel, and all aspects of the hardware.
	A bug in any one of those functionalities can prevent the proper functioning of the process.
	In contrast, in Composite, those component services depended on by a given component, and transitively depended on by them, the kernel (which is <10K LoC), and all hardware accessed by any of those components constitute the TCB.
	As only those component services that are required for the functioning of the component are depended on (principle of least privilege in action).
	Hopefully this analysis alone answers the question why microkernels are common in security sensitive domains.
- **Defense in Depth.**
	This is concerned with, intuitively, encouraging the construction of as many barriers as possible, over which attackers need to jump.
	Modern browsers have been great examples of this.
	Different tabs are separated out into separate processes to prevent compromised webpages from accessing other webpage data (barrier 1).
	The code executing in these processes executes within a software-provided sandbox that prevent access outside of valid ranges of memory (barrier 2).
	The javascript VMs prevent programming practices that access memory outside of the allowed objects (barrier 3).
	The [same-origin policy](https://en.wikipedia.org/wiki/Same-origin_policy) prevents requesting web resources outside of the original domain (barrier 4).
	The set of system calls that a browser tab can access is (or should be) restricted (barrier 5).
	The list goes on and on.
	Any one of these layers can (and will) have bugs that will be exploited.
	However, the likelihood that *all* of them will be exploited is quite unlikely.
	Outside of state actors, it is rare to have exploits that span many different layers of defense in depth.

	Note that defense in depth should inspire some sadness.
	It is a strong acknowledgement that there *will be compromises* of our code.
	It is inevitable.
	Given this, lets make it harder to effect widespread threats to CIA by adding more layers of checking and security.
	Put another way, there is an active *nuclear escalation* between those implementing the multiple layers of security, and those attempting to compromise each successive layer.

## Security Examples

Lets go through a few example systems.
How do they approach providing a capable and useful *refmon*, while designing the system fundamentally for strong security?
I'm going to focus on a *module-based description* of each system.
Focusing on modules, interactions between modules, and isolation between modules.
This should clarify how each system interacts with the security principles.
For example, if an application module can invoke many other modules (than it requires), that is not optimizing for the principle of least privilege; if two application modules depend on and can invoke the same module, that is not emphasizing least shared mechanism; a functionality consisting of multiple modules that is implemented in a single protection domain has increased separation of privilege if, instead, its modules are mutually isolated (and have different dependencies -- thus privilege); and the total complexity of all depended on modules (and modules they depend on transitively), along with hardware, defines the TCB.

### Microcontroller RTOS

Most RTOSes are designed to be quite modular.
RTOSes are often meant to run on microcontrollers with between 8 and 128 KiB of SRAM.
Your hello world programs likely are quite a bit larger than this.
To scale down, when necessary, these systems must be able to enable and disable different features and modules.
[FreeRTOS](http://aosabook.org/en/freertos.html) is a strong example of this type of system.

Though the OS consists of multiple modules, and multiple application (modules), it is common for them all to be in the same protection domain.
This enables extreme optimizations by using interrupt hardware for context switches, and removes the overheads of protection domain switches (and dual-mode switches) from interrupt and application handling paths.
In short, these systems traditionally focus on getting the maximum productivity out of every cycle, and protection didn't cut it as an excuse to add inefficiency.

This aversion to the overheads of protection wasn't, perhaps, as crazy as it sounds.
Embedded systems often existed within a controlled ecosystem.
A company (or contractors for that company) spent a lot of time integrating all of their systems together.
For commodity devices, the systems are often pretty self-contained.
*However*, now that we're hooking all of our IoT devices up to the network, it is clear that trading away security is no longer a reasonable trade-off.
This changes the calculus: now systems don't need to only be tested for integration into the surrounding environment, they must also be tested with *every possibly malicious input from the Internet*.
As you might imagine, this is not possible, nor easy to approximate.

*refmon analysis.*
The *refmon* is neither tamperproof, nor imbued with complete mediation.
As all code is run in kernel-mode, any of it can execute sensitive instructions for reprogramming the Flash to rewrite the *refmon*, or simply do a `memset(sram_addr, 0, sram_sz)` to all of SRAM to corrupt the *refmon*'s data-structures.
Complete mediation also doesn't hold:
Multiple applications are typically linked together to create the blob that is deployed on the embedded system.
This enables symbols defined by one to be accessed from another.
Even if you prevented such link-time, cross-application module references, applications can synthesize cross-module references trivially (`int *ptr = (int *)0xDEADBEEF` gives you access to whatever's at the address `0xDEADBEEF`).

*Security principles analysis.*
RTOSes with no facilities for isolation do focus quite strongly on *Economy of Mechanism*, and on a *small TCB*.
The RTOSes are able to avoid all of the complexity that comes with providing protection domain abstractions.
However, if this level of simplicity doesn't enable you to *verify all of your code*, then the system is not acquiescing to the security wisdom: assume that your system has bugs, and will be compromised.
Practically, I don't know of any RTOSes without isolation that are formally verified, so the arguments for Economy of Mechanism and small TCB are not convincing.

| Principle                                          | Traditional RTOS |
| ---                                                | ---              |
| **Complete Mediation**                             | Nope             |
| **Economy of Mechanism** & **Minimal TCB**         | Sure!            |
| **Least Common Mechanism**                         | Nope             |
| **Least Privilege**  & **Separation of Privilege** | Nope             |
| **Defense in Depth**                               | Nope             |

We can understand why the world of IoT is in a dire position.
Most RTOSes are fundamentally not equipped to handle malicious (or buggy) code.

### Separation Kernel

If you were to design a system with a primary goal to support two sets of applications -- *two domains* -- that cannot interfere with each in any way, how would you do it?
You might start by splitting system memory between the domains.
Going further, you might separate the I/O devices between the two domains.
Then for CPU management (on a single core system), you might make the scheduling deterministic -- for example just switching back and forth between the domains every quantum (e.g. every 10ms).
This is a simplistic rendition of the core of a [separation kernel](https://en.wikipedia.org/wiki/Separation_kernel).

The confidentiality and integrity of such a system is just about as strong as you can get^[Unfortunately, you have to also compensate for shared hardware resources. For example, a domain's careful manipulation of the shared cache can leak some information on context switches between domains (as the other domain can detect if a memory access is a miss or a hit from how long it takes). To compensate for this, the *refmon* must flush shared caches on context switches which negatively impacts performance.].
They effectively share no resources, so there are no means to access secrets in the other domain, nor corrupt resources accessed by the other domain.
Most system complexity exists in the domains.
More modern separation kernels (such as [Muen](https://muen.codelabs.ch/muen-report.pdf)) execute each domain as a VM using hardware support for virtualization to enable the VMs to manage their given resources.
The kernel of the system has a very simple job: boot the system, create a partition (of memory, I/O) per domain, and program the timer, and periodically switch between domains.
Devices are accessed directly from within each domain's VM, so the kernel doesn't even need to provide I/O facilities after bootup.
Strikingly, separation kernels can avoid even having system calls!

The (huge) downside of separation kernels is that they use complete partitioning of resources.
No I/O device can be shared between domains, memory is statically split between domains -- even if one domain wants more and the other isn't using it, and CPU is deterministically multiplexed -- even if one domain is idle.
In this way, separation kernels trade efficient resource utilization for simplicity and security.

*refmon analysis.*
As the kernel can be implemented to be free of user-level input (no system calls!), it becomes very difficult to tamper with the *refmon*.
Additionally, the *refmon* sets up initial domain access to resources, an ensures that those sets of resources are disjoint, and (in Muen, for example) ensure that hardware is set up such that dynamic resource requests only access the assigned sets of resources (e.g. enforced by page-tables).
Thus, the *refmon* effectively performs complete mediation on resource access.

*Security principles analysis.*

| Principle                                          | Separation kernel                                                                                                                                                                                                                                                                                                                |
| ---                                                | -------------                                                                                                                                                                                                                                                                                                                    |
| **Complete Mediation**                             | All resources are partitioned at boot time, and access is controlled by hardware such as page-tables.                                                                                                                                                                                                                            |
| **Economy of Mechanism** & **Minimal TCB**         | The separation kernel is about as simple as you can make it while still providing strong isolation. The TCB is small and includes only the simple *refmon*, and hardware.                                                                                                                                                        |
| **Least Common Mechanism**                         | The separation kernel *refmon* itself is the only shared mechanism between domains, and it tries to be pretty simple.                                                                                                                                                                                                            |
| **Least Privilege**  & **Separation of Privilege** | The separation kernel doesn't provide many abstractions itself, and within each domain, the it exerts no control over resource access. Similarly, separation of privilege requires communication between modules, and communication between domains is prevented! As such, they don't provide particularly strong PoLP, nor SoP. |
| **Defense in Depth**                               | Defences within a domain are up to the logic within the domain.                                                                                                                                                                                                                                                                  |

Separation kernels are a great example of exceedingly strong security, while trading off usability (no sharing!), and efficient resource utilization (*static* partitioning between domains).

### UNIX

UNIX's filesystem security model is based on permissions for users and groups, and includes read, write, and execute access.
Both files and directories have the specified permissions settings.


How can we analyse security within a monolithic system?
When assessing a process's capabilities for accessing resources, and triggering potential bugs in the *refmon*, we should think of system accesses in two dimensions:

1. Which system APIs (thus modules) -- which system calls -- do we have access to?
2. Which resources do we have access to through system namespaces (i.e. arguments to identify resources in the APIs)?

The PoLP is concerned with both.
[OpenBSD](https://www.openbsd.org/papers/cuug2019-predictable.pdf) has a huge number of defense in depth approaches to make it more difficult to 1. take over a process, and then 2. do anything useful if you do.
Notably, it has a set of system calls that address these two dimensions of resource access:

- `pledge` enables a process to revoke privileges from itself by removing *classes* of system calls from future invocation.
	These classes include reading or writing to the filesystem, accessing the network, doing simple `stdio` calls (read and write to open file descriptors), etc...
- `unveil` enables a process to refine which parts of the file system it should be able to access (and what type of access).
	It can refine its access to only small subsets of the FS, and, finally, call `unveil(NULL, ...)` to "lock in" the restricted access.

Both of these let a process implement some self-inflicted PoLP.
Note that this is smarter than you'd think: a server can `fork`, call `pledge` and `unveil` to restrict the child to only the resources it requires, then call `exec` to execute a program.
A perhaps more common usage pattern is to perform initialization (including opening many connections and files), then use these functions to restrict post-initialization execution.
The assumption is that compromises will occur *after* initialization, thus that is the important execution phase to restrict using the PoLP.

A very important implication of these APIs is that they are *discretionary*.
Discretionary security mechanisms are used at the volition of a program or user, instead of enforced by system-wide policies.
We'll revisit this soon.

#### [Discretionary Access Control (DAC)](https://en.wikipedia.org/wiki/Discretionary_access_control)

UNIX was created around DAC.
That is to say that users and programs have the ability to change the permission of many of the resources that they already have access to.
This is very convenient.
If you want to share a file with a friend on a shared server, you can simply `chmod`/`chown` it to give them access.
But this means that the overall security of the system is dependent on the decisions made by users and, more importantly, the programs that execute with the user's credentials.
A bug in any such application can make any resource more broadly available (think: ssh keys, passwords, etc...).

#### [Mandatory Access Control (MAC)](https://en.wikipedia.org/wiki/Mandatory_access_control)

In contrast to security that requires the user/program to not incorrectly create security vulnerabilities, MAC is concerned with creating system-wide policies for restrictions on CIA that users cannot subvert.
Generally these policies are often viewed as restricting information flow (Confidentiality -- e.g. [Bell-LaPadula](https://en.wikipedia.org/wiki/Bell%E2%80%93LaPadula_model) security model ), or on the ability to modify (Integrity -- e.g. [Biba](https://en.wikipedia.org/wiki/Biba_Model) security model).
I won't go into the details of these example policies as they are generally not very practically useful.
See [this video](https://www.youtube.com/watch?v=IAnggHc0_aA) for a brief overview of Bell-LaPadula.

Why aren't these policies practical?
Lets use the intuitive example of confidentiality levels of documents and people.
What if the president wants to tweet?
That would never be allowed by one of these policies.
What if a sysadmin wants access to logs for security-sensitive process?
What if users want to collaborate (google docs)?
None of these would generally be allowed given these strict policies.
So what do practical systems do?
They often apply these policies, but only to the programs and data that have the highest likelihood or impact of compromise, and well-formulate and assess any exceptions that are required (the government process for this is called [redaction or santization](https://en.wikipedia.org/wiki/Sanitization_(classified_information))).

Regardless these specific policy's practicality, it is important to have MAC enforcement in the system to avoid the whims and bugs of users and programs to threaten full-system security.
MAC implementation in Linux is possible using [*Linux Security Modules*](http://www.cse.psu.edu/~trj1/cse544-s10/papers/lsm.pdf).
*Security modules* (in the kernel) get *callbacks* on *all* resource accesses (e.g. even on `read`/`write`, not just on `open`), and can make yes/no decisions (which supports *revocation*).
This enables different security policy *modules* to define different MAC (or stronger DAC) policies.

- LSM is based on the [Flask security architecture](https://www.cs.cmu.edu/~dga/papers/flask-usenixsec99.pdf).
- Details about [LSM](https://www.kernel.org/doc/html/latest/security/lsm.html) and its [API](https://www.kernel.org/doc/html/latest/security/lsm-development.html).

	- After DAC access control decisions are made, if they allow access, then the MAC access control decisions of LSM are executed.
		Each is a simple callback function into the security module for the resource and principal.
	- Each object (e.g. files, network ports) is associated with a set of labels.
	- Each principal (e.g. user, process) is associated with a set of labels.
	- The module determines which principals can access which labels.

Lets look at a few of the security modules.

*MAC on Linux using pathnames.*
Simple ([apparmor](https://en.wikipedia.org/wiki/AppArmor), [Tomoyo](http://tomoyo.osdn.jp/2.6/index.html.en)): label objects, identified by [*pathnames*](https://lwn.net/Articles/277833/), that a principle can access with the same label.
But what if an intermediate principal provides *indirect* access through IPC?
*Policy file* for each application specifies accessible paths (and `rwx` permissions within those paths).
This approach is based on *whitelists* -- assuming that we have no access to resources, and only specifying paths for which we should have access^[In contrast, blacklist based approaches assume [ambient](https://genodians.org/guido/2019-03-26-on-ambient-authority) [authority](https://en.wikipedia.org/wiki/Ambient_authority), and the paths would specify which a principal *cannot* access.].

These approaches, in many ways, similar to `unveil` (discussed above for OpenBSD): both specify paths that an application can access.
The difference is that the parent process that will create the new application process (via `exec`) must manually include the proper policies for how to `unveil` that application.
Importantly, one can run the application *without* calling the `unveil` API.
In contrast, the LSM approaches specify policy files and the kernel (as part of the TCB), and *any* execution of the application must adhere to that policy.
In this way, it is a mandatory limitation on access of the application.

*MAC on Linux via labels and information flow -- seLinux.*
Complex (seLinux[(1)](https://en.wikipedia.org/wiki/Security-Enhanced_Linux)[(2)](https://wiki.centos.org/HowTos/SELinux)): controlling [information flow](https://wiki.gentoo.org/wiki/SELinux/Information_flow_control) -- principals accessing objects gain their labels (and vice-versa), and you can only access a specific set of labels (often there is a total order of labels like classification levels).
The intermediate principal gains the label of the information you shouldn't access, thus preventing IPC.
This can *dynamically* propagates labels through a system as resources are accessed by different principles.
Policies can determine which sets of labels are allowed to access resources with which sets of labels.
If user *A* is not allowed to share any information with user *B*, and *A* writes a file *f*, that file might receive the label for *A*.
Later, if *B* attempts to accesses the file, the policy would disallow the action.

Unfortunately, the policy files for seLinux are huge and complex, which puts them outside of the grasp of normal users and developers.
Android makes extensive use of seLinux.
This is because the policies can be determined by the platform developers (of the Android OS), and not by application developers, and they focus on preventing applications from accessing system resources (contacts, etc...) and other apps.

*refmon analysis.*
The kernel is "in the loop" in the use of all resources.
Most are mediated by system calls, through which the complex abstractions provided by the kernel are manipulated.
Though `load`/`store`s to memory don't go through the kernel (thanks page-table walking hardware!), but the kernel *sets up the access to physical memory*.
Similarly, "kernel bypass" techniques for I/O enable drivers to operate in user-level and directly talk to devices (recall the Demikernel).
Even in this case, the kernel uses memory mapped I/O to carefully map the device into user-level, and only then can the application interact with it.
Thus, the kernel certainly provides complete mediation.

However, it is not tamperproof.
Most kernels (OpenBSD excepted) enable kernel code to be added at runtime in the form of "kernel modules".
This typically only requires `root` access (thus why `root` is often synonymous with kernel access.
Some techniques will only allow modules to be loaded at boot-time, but other glaring holes exist.
With sufficient permission `/dev/mem` can be written to overwrite physical memory.
More importantly, the history of vulnerabilities in the Linux kernel demonstrate that there will always be bugs in the kernel, many of them exploitable.
Thus it is prudent, when security is important, to not assume that you can trust the tamperproof-ness of the kernel.

*Security principles analysis.*

| Principle                                          | UNIX                                                                                                                                                                                                |
| ---                                                | -------------                                                                                                                                                                                       |
| **Complete Mediation**                             | Yes, the kernel is responsible for making resources accessible                                                                                                                                      |
| **Economy of Mechanism** & **Minimal TCB**         | Monolithic systems have a *lot* of *complex* code in the TCB                                                                                                                                        |
| **Least Common Mechanism**                         | The entire kernel is shared mechanism, which is not small.                                                                                                                                          |
| **Least Privilege**  & **Separation of Privilege** | Applications can be broken into multiple processes, each with separate access to resources. `systemd` is actually an example of some of this in action, as is the architecture of the `ssh` server. MAC mechanisms attempt to more finely tune the access rights of applications but are always fightening against default ambient authority. |
| **Defense in Depth**                               | Countless techniques to compensate for specific attacks, detect potential attacks (antivirus), and "harden" codebases (W^X, stack hardening, ASLR, Application/kernel relinking, etc...).                                                                              |

Note that this analysis doesn't change much for containers.
Defense in depth is strengthened as containers cannot access each other's namespaces, but the fundamental problem is the liability of shared functionality in the kernel, and containers don't change that.
To take existing systems, and add more depth to the security, containers are useful, but they are not fundamentally a technology targeted at increasing system security.

We've seen how MAC and defence in depth are added to monolithic systems.
This is an attempt to drag systems -- based on DAC and that have *refmon*s that, for all intents and purposes, should be considered tamper-able -- toward heightened security.
This adds a lot of complexity (increasing TCB)

### VMs

Lets discuss a few definitions.
There is some confusion about these in the literature, so I'll try and be explicit about what I mean here.

- We'll assume the use of *Hardware Virtualization Acceleration* (HVA) (e.g. Intel's VMX) which enables the avoidance if exceptions for VM page-table management during (but not limited to) *nested page-tables* (e.g. Intel's EPT), and dual-mode protection including a significant fraction of the sensitive instructions.
	With nested page-tables, the VM's kernel maps between virtual and VM-physical.
- The *hypervisor* is the kernel-resident set of modules that manages VMs, at least providing a mapping between VM-physical addresses down to the machine's actual physical addresses (i.e. with page-tables), scheduling of VMs, and interrupt virtualization.
	In short, the hypervisor has the logic to execute a VM, uses hardware virtualization acceleration to avoid needing to service many VM exceptions, and coordinates between the VM and the VMM.
- The *Virtual Machine Monitor* (VMM) is the set of modules in the system that coordinate the boot-up and teardowns of VMs, manage their memory, and provide I/O services to the VM.
	This last function -- providing I/O -- is, in many ways the most difficult and sensitive.
- A VM uses *paravirtualization* when it relies on *software abstractions* to communicate with the surrounding system (e.g. to conduct I/O, rather than pretend it is talking to an I/O device).
	A VM can make a *hypercall* to request software services from the hypervisor.
	It is, in many ways, just a fancy name for a *system call*, except that it is a trap from a VM to the hypervisor.

There are many other important concepts such as layers below the hypervisor, and specific I/O devices that aid in virtualization to enable a single physical device to be *directly interacted with* from multiple VMs (see [SR-IOV](https://en.wikipedia.org/wiki/Single-root_input/output_virtualization)).

I'm going to distinguish between three different types/classes of VM systems:

1. *Monolithic kernels* such as Linux that provide a VM infrastructure (e.g. via `kvm`/`qemu`).
	Linux is the hypervisor, and `qemu` (or [Firecracker](https://firecracker-microvm.github.io/)) is the VMM.
	`kvm` is the Linux abstraction for HVA.
	Much of I/O subsystem uses Linux's abstractions.
	`qemu` provides emulation of the device, which uses the kernel's I/O.
	Paravirtualized I/O simply reduces the burden of the device emulation as both the VM, and the VMM understand the device is a software construct (see [`virtio`](https://wiki.libvirt.org/page/Virtio) for an example).
1. *Specialized hypervisors* that export device drivers to a VM.
    In contrast to the monolithic approach, systems such as Xen do not include the device drivers in the hypervisor.
	Instead, they use paravirtualized I/O to make requests from VMs to a special VM with direct access to devices.
	This VM receives these requests, and forwards them to the real device drivers.
	This has the benefit of simplifying the hypervisor, but puts a lot of privilege in the special VM.
1. *Minimal hypervisors* that provide IPC between VMs and VMM mainly for shared I/O services.
	Microkernels such as [Nova](https://hypervisor.org/) transform VM exceptions into IPC, and use the optimized communication facilities to implement the VMM in user-level.

*refmon analysis.*
Assuming that the hardware provides facilities for virtualization acceleration (e.g. Intel's VT-X), hypervisors can be quite simple^[Without such support, they must emulate paging using *shadow paging*, emulate dual mode protection system calls and exceptions, and support *trap and emulate* of sensitive instructions. This is complex and ends up taking quite a bit of code.].
This can be seen in the design of Nova, which is essentially a microkernel with small additions to support VMs.
Lets go through the classes of systems, one by one.

**Monolithic kernels** that support virtualization spread their *refmon* across the hypervisor, which is large and complex, and the VMM.
The latter has historically been very complex (`qemu`), but there are projects to simplify it down to the bare minimum (e.g. firecracker).
The general integrity of the hypervisor (Linux kernel) is the same as the monolithic analysis in the previous section.
*However*, the `kvm` API does *not* allow VMs to make direct system calls, thus indirecting service requests through the VMM.
Given this, the kernel is most likely protected to the extent that the VMM is protected.
The VMM is often quite complicated, and when performing I/O emulation, has a [history of compromises](https://www.cvedetails.com/vulnerability-list/vendor_id-7506/cvssscoremin-7/cvssscoremax-7.99/Qemu.html).
This better justifies why firecracker and related approaches seek to in many cases remove device emulation (instead using paravirtualized devices), and only use a restricted set of devices.
That said, if the VMM is compromised, then the kernel is the next target for further privilege escalation, and defense in depth approaches might be used to try and slow down attackers.

In summary, the attack surface (APIs) that a VM has access to is relatively restricted.
This has the practical benefit of making compromises more challenging, and not exposing the full kernel syscall API without further escalation.

**Specialized hypervisors** can be smaller^[Xen is on the order of 100K LoC.] as they don't require I/O drivers and subsystems.
They do often provide an API of [around 20](http://www-archive.xenproject.org/files/xen_interface.pdf) [hypercalls](http://xenbits.xen.org/docs/unstable/hypercall/index.html) that VMs can use to ask for specialized services.
This does constitute an attack surface of the hypervisor, and it has [been](https://googleprojectzero.blogspot.com/2017/04/pandavirtualization-exploiting-xen.html) [exploited](https://www.blackhat.com/docs/us-16/materials/us-16-Luan-Ouroboros-Tearing-Xen-Hypervisor-With-The-Snake-wp.pdf) in the past.
Additionally, the driver VM is yet another attack target.
As this VM is typically a fully-featured Linux instance, escalation after an attack is simpler.
The attack surface of the VM is often through the I/O subsystems through paravirtualized devices.

The comparison in security between Xen and Linux is quite nuanced.
In short, both have a long history of vulnerabilities.
For Xen, see the [combination of attacks](https://xenbits.xen.org/xsa/) on the  hypervisor, and on the special driver VM.
In one architecture (Linux + firecracker), the hypervisor is monolithic and large, and the VMM can be smaller and more focused.
In the other (Xen), the hypervisor is simpler, but the VMM is monolithic and large.
In both there is a pathway to attack the monolithic system, and escalate privilege.
Also in both, the challenge in doing so is likely enough for all but the most sensitive systems to dissuade most attacks^[However, if the technology is the uniform foundation of a cloud infrastructure, then the up-side of a zero-day might be appealing to some attackers.].

**Minimal hypervisors** based on microkernels expose a small API to VMs, often focused on performing IPC to communicate with a VMM.
Nova is enlightening here to illustrate the complexity involved here: the hypervisor is around 10K LoC, while the VMM (running in a user-level process) is around 30K LoC.
These systems are architected to have a VMM per-VM.
This is paired with a minimal interface of the VMM to access other system services and resources to severely limit the ability to escalate privilege to the point where the CIA of other VMs is threatened.
Each VMM *does* have some access to device drivers, but usually through IPC to a shared device driver, thus adding additional barriers to escalation.

*Security principles analysis.*
Here, **MHV** means Monolithic Hypervisor, **SHV** means Specialized Hypervisor, **$\mu$HV** for microkernel Hypervisor.

| Principle                                          | VMs                                                                                                                                                                                                                                                                                                              |
| ---                                                | -------------                                                                                                                                                                                                                                                                                                    |
| **Complete Mediation**                             | MHV, SHV: the hypervisor and VMM interact to constrain the set of resources accessible to the VM. $\mu$HV constraints a VM's access using the simple hypervisor, including IPC to the VMM.                                                                                                                                                          |
| **Economy of Mechanism** & **Minimal TCB**         | HVM, SHV: the total size of the TCB is very large (larger than a simple monolithic system as it also includes the VMM). SHVs can be argued to have even larger TCBs than HVMs as they must include the external hypervisor. $\mu$HVs require only the minimal hypervisor, the small-ish VMM, and whatever device drivers are necessary.                                                                                                                |
| **Least Common Mechanism**                         | SHV systems tend to share the driver VM, and the hypervisor between VMs. In contrast, modern MHVs can have a separate VMM instance per VM (different `qemu`s instances), but share the large monolithic OS. $\mu$HV shares only the hypervisor, and the necessary shared I/O services. If two VMs don't use shared I/O devices, the only shared mechanism between them should be the hypervisor (10K LoC!). |
| **Least Privilege**  & **Separation of Privilege** | VM infrastructures enable separate VMs to be configured with separate access to memory, and preemptive access to the CPU. All of the VM infrastructures discussed above separate the hypervisor from the VMM, and at least $\mu$HV and MHV support separate VMMs per-VM.                                                                                                                                           |
| **Defense in Depth**                               | None of the discussed infrastructures have the defence in depth of protection within a protection domain we saw in Linux. This is mostly simply because these infrastructures provide quite a bit more isolation and have smaller attack surface areas. $\mu$HVs to add isolation of the minimal I/O services.                                                                                                                                                                                                                                                                                                                 |
### Capability-Based OS

I won't go over CBOSes in much detail here as we've discussed them in detail earlier in this book.
In short, they are designed around the PoLP by providing fine-grained capability-mediated access to only the resources a module requires, and isolation can be provided on a module-granularity (priv. sep.).
The focus on a minimal kernel with IPC as the dominant mechanism for service isolation and module coordination.

Capability-based systems naturally provide strong [confinment](https://www.cs.utexas.edu/~shmat/courses/cs380s_fall09/lampson73.pdf), but are not a very natural for *complicated* MAC policies!

- Example: If you provide a capability to a domain, how can you ensure that it is read-only shareable with higher-privilege/classification domains, and write-only shareable with lower-privilege/classification domains (Bell-LaPadula)?
	Policies need to be layered on top of the raw capability delegation and revocation policies of the underlying system.
- Composite defines sharing policies in user-level, thus avoiding this challenge, but L4 variants must build quite a bit to provide such MAC policies.



- Persistency and capabilities

*refmon analysis.*
*Security principles analysis.*

| Principle                                          | CBOSes        |
| ---                                                | ------------- |
| **Complete Mediation**                             | CBOS kernel mediates all resources accesses either via page-table or capability-table lookups. Resources defined in components require their own logic to determine which client should access what abstract resource.             |
| **Economy of Mechanism** & **Minimal TCB**         | The kernel is very small (<10K LoC), and only those services that an application requires for its service should be in its TCB.              |
| **Least Common Mechanism**                         | The kernel is shared between all modules, but is small. Only those user-level services required by an application are part of its dependencies *and* potentially shared with other applications.              |
| **Least Privilege**  & **Separation of Privilege** | Systems are encouraged to split modules across protection domains, and (through the capability model) allow each protection domain to have access only to those resources they require. These systems are, in some ways, designed for least privilege and the separation of privilege. In practice, some of these systems end up creating large monolithic components, rather than split up the modules across protection domains.              |
| **Defense in Depth**                               | CBOSes encourage many barriers between modules, but don't beyond that focus extensively on defence in depth.              |

## Mechanisms (TODO)

> This section is TODO

### Defense in Depth in Conventional Systems

- Linux Security Modules ([LSM](http://www.cse.psu.edu/~trj1/cse544-s10/papers/lsm.pdf) and [its](https://www.kernel.org/doc/html/latest/security/lsm.html) [API](https://www.kernel.org/doc/html/latest/security/lsm-development.html)) try to simplify the reasoning behind the *refmon* in Linux.
- LSM "hooks" (what are "hooks"?) for reference monitor implementation

	- [pathname](https://lwn.net/Articles/277833/)-based security in [Tomoyo](http://tomoyo.osdn.jp/2.6/index.html.en),
	- [AppArmor](https://wiki.ubuntu.com/AppArmor) and see an example of an AppArmor [webserver access policy](https://gitlab.com/apparmor/apparmor/-/blob/master/profiles/apparmor.d/usr.sbin.apache2), and
	- [Smack](http://schaufler-ca.com/description_from_the_linux_source_tree) and other [related](https://lwn.net/Articles/244531/) documentation

- [KPTI](http://www.brendangregg.com/blog/2018-02-09/kpti-kaiser-meltdown-performance.html) analysis

	- Linux vs. cos w/o KPTI vs. cos w/ vs. cos with per-component mappings

- The extent of complications for [complex kernels](https://lwn.net/Articles/749707/)

- The complexities in *refmon* implementation are many.
	Beyond traditional software correctness, one needs to worry about Time of Check to Time of Use ([TOCTTOU](http://www.watson.org/~robert/2007woot/)) bugs that open up small windows of *refmon* incorrectness.
	An amazing example: [dirtyCOW](https://dirtycow.ninja/).

- dbus $\to$ kdbus [concerns](https://lwn.net/Articles/649111/) and [benefits](https://lwn.net/Articles/640360/) clients close buffers, clients cause server to block, memory consumption for published messages, "By moving broadcast-transmission into the kernel, we can use the time-slices of the sender to perform heavy operations", ...

### Reimagining "Users" in UNIX

Since *refmon* decisions are made relative to users, having users be literally human users of the system, collects together a lot of data, executables, and functionality into the same principle.
Instead, engineers have started reimagining users to be closer to specific services and applications in the system.
Examples of this:

- [systemd](http://0pointer.net/blog/dynamic-users-with-systemd.html) services running as their own user
- Though Android is a flavor of Linux, the [Android Platform Security Model](https://arxiv.org/pdf/1904.05572.pdf) is quite a departure from typical techniques.
	Applications are security principals.


How should we design the *refmon*?

Taxonomy

- Access control matrix
- Information flow control
- Access Cntl Matrix protection vs. Information flow control

    - IFC: Flume, seLinux [policies](https://github.com/SELinuxProject/refpolicy/wiki)
	- ACM: Capability-based OSes, [Capsicum](https://www.cl.cam.ac.uk/research/security/capsicum/)
	- Nexus

Kernel minimality and design

- core of the security requirements
- separation of mechanism and policy ([chapter 11](https://ocw.mit.edu/resources/res-6-004-principles-of-computer-system-design-an-introduction-spring-2009/online-textbook/part_ii_open_5_0.pdf))

- [KPTI](http://www.brendangregg.com/blog/2018-02-09/kpti-kaiser-meltdown-performance.html) analysis

	- Linux vs. cos w/o KPTI vs. cos w/ vs. cos with per-component mappings

- VMs and related tech

	- hypervisor implementation via `libvirt` & [firecracker](https://www.youtube.com/watch?v=cwruf1ERAKM&list=TLPQMjYxMjIwMjCdrIWpLWTXvg)
	- Analyse the concrete [protection provided by gvisor](https://gvisor.dev/docs/) (read the entries in the "Architecture Guide" on the left)
	- [Nova](http://hypervisor.org/eurosys2010.pdf) ([github](https://github.com/udosteinberg/NOVA))

- Capability-based OSes

	- Composite
	- [Grant](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.130.9896&rep=rep1&type=pdf)-[take](http://courses.cs.vt.edu/~cs5204/fall99/distributedSys/groener/takegrnt.html) security model
	- [seL4](http://sigops.org/s/conferences/sosp/2013/papers/p133-elphinstone.pdf)
	- [Nova](http://hypervisor.org/eurosys2010.pdf)

- Adapting existing systems

	- seLinux
	- containers
	- dbus $\to$ kdbus [concerns](https://lwn.net/Articles/649111/) and [benefits](https://lwn.net/Articles/640360/) clients close buffers, clients cause server to block, memory consumption for published messages, "By moving broadcast-transmission into the kernel, we can use the time-slices of the sender to perform heavy operations", ...
