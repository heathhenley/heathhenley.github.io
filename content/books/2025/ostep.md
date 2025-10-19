---
title: "Operating Systems: Three Easy Pieces"
draft: False
hideDate: true
---

By Remzi H. Arpaci-Dusseau and Andrea C. Arpaci-Dusseau

This is the quintessential OS book that's widely recommended and used for many
OS courses. I realized I had never picked it up - and given that my path to SWE
was a bit un-conventional, I still have a lot of gaps - this was a great way to
fill them in, clear some things up and reinforce things I've forgotten long ago.

It's not too low level, but it's got a lot references to jump off from to dig
deeper as needed.

These are scattered notes from reading the book - level of detail varies depending on how interesting I found the topic, how much I already knew, and how much I felt like taking notes at that time.

## Part 1 - Virtualization
The virtualization layers over the actual hardware 
### CPU Virtualization 
Process and process api - kind of an introduction
- Fork/exec to create process
- Kernel vs user mode
- System calls, sensitive stuff, mem reads etc all in kernel mode 
- How the CPU allows processes to run arbitrary code, but keeps control 
- TRAP table loaded at boot time with all system calls - so no one can mess with it

Context switching
- Interrupts periodically or io (needs system call) CPU decides whether to context switch or not
- Load instructions / registers / current pointer from memory

Scheduling methods
- Simple Impractical - round robin, shortest first, shortest to complete first, etc
- Optimize turnaround time - the time between getting queued and completing 
- Optimize response time - the time between getting queued and first being scheduled 
- Actually used in some form - Multi Level Feedback Queue

Multi-level feedback queue
- Multiple queues (high priority to low priority)
- Jobs in highest run before jobs in any lower queues
- If they have the same priority they run in round robin
- All processes start with highest priority 
- As they use “allotments” they are moved down
- All processes moved up to highest sometimes

More scheduling: proportional share
- Optimize “fairness”
- Lottery - user assigns tickets to convey priority, processes randomly selected based on tickets to run —> not obvious how to assign, for shirt running processes not always very “fair”, random 
- Stride - deterministic version but similar to lottery, needs some extra state and to handle adding in new processes in a special way 
- “Completely fair scheduler” - CFS - as process runs it accumulates virtual runtime, always run the process with the lowest virtual runtime - priority can be adjusted using nice 
- CFS - store running processes in a red black tree for search, insert, remove trade off
- CFS - processes that sleep frequently might never get their fair share of runtime, when it wakes up it’s runtime is set to the lowest value in the rb tree (so it won’t  be very behind and starve the other processes until catching up)
- Maybe not as good for user systems, but good for something like a data center
- “Nice” ness controls priority on Linux based systems

Scheduling with multiple cpus
- Caching 
- Cache coherence in concurrent systems is a problem - “bus snooping” one option, caches watch changes and if a value they have is updated, they invalidate or refetch 
- Cache affinity - if a process has previously run on a CPU, it should run on that one again if possible because some of the state / data it needs might will likely already be in the programs cache 
- Single Q scheduler- reuse the framework for a single CPU - instead of next job grab next n jobs. Biggest issue is cache affinity, jobs will flip plop around and run on different processors, but can be handled by modifying algorithm to account for affinity. Synchronization also introduces overhead, they need to lock to access the shared queue 
- Multi Q scheduler - each CPU with its own queue and jobs assigned randomly or by some heuristic. Fixes cache affinity and locking - but new issue is load imbalance….
	- “Work stealing”  - CPU with low queue looks at another queue and migrates jobs if it’s more full than it is. Requires balance - can’t look too much or pay synchronization price, can’t look too little or workload is imbalanced 
- Actual Linux schedulers: O(1), BF scheduler, CFS (completely fair scheduler)

### Memory virtualization
OS needs to provide abstraction layer for running processes code, stack and heap - address space - every address you can see as a user is virtual s virtual s virtual 
- needs to be efficient, restrict programs to only their address space, but also appear to the program as if were physical memory (transparent)
- process address space organized with stack and heap at opposite ends so they can grow - actually program is static
- process address space organized with stack and heap at opposite ends so they can grow - actually program is static

C memory api interlude
- malloc/free - calloc, realloc 
- common mistakes, mem leaks, dangling ptr, use after free, etc
- tools valgrind, etc
- brk and sbrk- only new part of this chapter for me, lower level api that malloc uses 

Address translation
- OS needs to get out the way and let process access memory but at the same time retain control and limit access only to its allotted memory
- Similar principles to “limited direct execution” in cpu virtualization 
- OS wants to stick each processes space in actual memory somewhere - uses hardware to apply offset so each process thinks address space starts at 0
- The part of the CPU responsible for this is called the Memory Management Unit (MMU)
- Simple base bounds - CPU has base and bounds registers
- OS sets them for running process (in kernel mode) so that virtual memory accesses can mapped to physical ones 
- Base is just an offset, bounds is a limit check to prevent out of bounds 
- CPU needs to be able to raise exceptions and probably terminate processes that try to access out of bounds memory or change base/bounds registers 
- OS can easily move processes around - suspend, copy the memory into new spot, update base/bound registers 
- Exception handlers for illegal access set up in kernel mode at boot 
- Simple base and bounds -> fragmentation - need to generalize to be less wasteful and support large address spaces 

Generalized Base and Bounds
- Use a base and bound param in the MMU for each segment of address space (program, stack, heap)
- The whole address space together is wasteful - this is more efficient, not big empty chunks of virtual address space 
- This called **segmentation** 
	- Trying to access invalid memory —>  segmentation fault! 
	- Term still used on machines with no segmentation support lol
- How does the hardware know which segment a virtual address refers to?
	- Could use top two bits (0 - code, 1 heap, etc) - or 1 bit and keep code and heap together
	- Could be implicitly done using what the address was based off of (program counter - code, base / stack ptr - stack, otherwise heap
- To support growing from the offset in either direction there’s another bit on the MMU for each segment 
- Support for **Sharing** - also adds a new register per segment in the MMU to describe permissions (read/write/execute)
	- Processes can now share code segments without it looking any different
	- Need to add permission checks to look ups of course 
- Some systems have fine grained segmentation vs the course grained approach here, same idea but needs a segmentation table
- On context switches, values need to be swapped for the current process (segment info, protection, etc
- Growing segments as process needs it (if possible)
- Managing free space and minimizing **external fragmentation** is an important issue
	- Lots of algorithms exist to track free space and determine allocation but not compact too much to make growing harder
- Main issues still w/ variable sized memory
	- External frag - can’t avoid, can only try to address
	- Not flexible enough to support a sparse address space - large sparse heap in one logical segment, then the whole segment needs to be in physical memory to be accessed 

Generalized Base and Bounds (**Segmentation**)
- Use a base and bound param in the MMU for each segment of address space (program, stack, heap)
- The whole address space together is wasteful - this is more efficient, not big empty chunks of virtual address space 
- This called **segmentation** 
	- Trying to access invalid memory —>  segmentation fault! 
	- Term still used on machines with no segmentation support lol
- How does the hardware know which segment a virtual address refers to?
	- Could use top two bits (0 - code, 1 heap, etc) - or 1 bit and keep code and heap together
	- Could be implicitly done using what the address was based off of (program counter - code, base / stack ptr - stack, otherwise heap
- To support growing from the offset in either direction there’s another bit on the MMU for each segment 
- Support for **Sharing** - also adds a new register per segment in the MMU to describe permissions (read/write/execute)
	- Processes can now share code segments without it looking any different
	- Need to add permission checks to look ups of course 
- Some systems have fine grained segmentation vs the course grained approach here, same idea but needs a segmentation table
- On context switches, values need to be swapped for the current process (segment info, protection, etc
- Growing segments as process needs it (if possible)
- Managing free space and minimizing **external fragmentation** is an important issue
	- Lots of algorithms exist to track free space and determine allocation but not compact too much to make growing harder
- Main issues still w/ variable sized memory
	- External fragmentation - can’t avoid, can only try to address
	- Not flexible enough to support a sparse address space - large sparse heap in one logical segment, then the whole segment needs to be in physical memory to be access

Managing free memory
- Not hard with fixed size chunks, gets interesting with dynamic chunks
- **Internal fragmentation** (missed this def before - if the allocator hands out bigger chunks than needed, that waste/unused spaces is “internal frag”
- Splitting / coalescing free list (linked list stuff from cs)
- Header with chuck size stored (that’s how free knows how big the chunk is)
- Free list embedded into actual memory - storing size and next free chunk location
- Growing the heap strategies all have trade offs / perform differently based on input 
	- Best fit - traverse and take closest to requested 
	- Worst fit - traverse and take biggest free chunk 
	- First fit - faster, takes the first free chunk that fits the requested size 
	- Next fit - extra pointer to where you last looked, spread search over the list a bit better but still similar perf to first fit 
- Other approaches 
	- Segregated lists - separate list for free chunks of an application specific size, allocation and freeing easy if working with a chunks of fixed size
		- how much do you allocate for these special requests versus a general allocator though?
	- Buddy allocation - designed around making coalescing simple - divide free space by two until the next division would make the chunk too small - on free, check buddy, if free coalesce, and recurse on that until buddy is not free 
- Major issue with all approaches so far is scaling - searching list is slow - most real allocators use more complex data structures (trees) to make searching more efficient 

Paging (Introduction)
- Memory virtualized into fixed size chunks (**page**) - memory viewed as a bunch of slots called **page frames**
- Keep a per process **page table** structure to map virtual address space pages for a process to physical memory 
- Virtual address top bits describe **virtual page number**, the lower bits are the offset into the page
- Page table translates page number to physical frame - offset is the same
- What’s in a page table entry?
	- Page frame number 
	- valid bit —> if the memory is in use by the process or just an empty part of the address space
	- Protection bits like before 
	- Present bit - on disk (swapped out) or in physical memory 
	- Reference bit / accessed bit - used to determine which pages are used a lot and should not be swapped 
	- Dirty bits - memory was modified since being read into memory 
- Better than segmentation for fragmentation 
	- Problem - translation - Extra memory load for each existing load (to load the page table entry) - so without hardware or other trick would be too slow 
	- Problem - size - Page table would also be too big, wasting memory on page table instead of programs - 64bit sys would be yuge

Paging - making address translations faster 
- **TLB**- **translation-look aside buffer** - really it’s an address translation cache
- Hardware cache in the MMU for common translations - hardware checks the TLB before going to the page table to get the page table entry, way faster 
- Bigger page size is better for cache hits 
- Temporal and spatial locality benefits - array elements etc stored near each other (chunks on same page) (spatial) - re-accessing data (temporal)
- TLB misses can be handled by hardware (CISC) or using software (RISC) - software is more flexible because page table can be any data structure - need to avoid triggering infinite recursion if the trap handler code for cache misses is in a virtual address not in TLB (sometimes translation is in there permanently etc to fix this, or handler code is permanently located in unmapped memory that doesn’t need a translation)
- TLB content is process dependent (actively running processes) - need to handle context switching 
	- Could flush it all on context switches - works but cost could be high, lots of misses 
	- Can add an **address space identifier** (ASID) to the TLB entry - on context switch a privileged register is set to the ASID of the running process (like PID but fewer bits)
	- Cache replacement policies - LRU - good but has poorly performing edge cases, eg always reading in increments of n+1 where n is the TLB size (miss every time) - random access doesn’t have this issue but doesn’t benefit from locality 
	- if the number of pages a program uses in a short period of time are larger than the TLB, there is a performance hit - **cache coverage** —> dbms like pg, can benefit from support for increasing page size 

Paging - making page table smaller 
An array / linear page table is too large (there’s one for each process) - what can we do about it?
- One approach: Combine segmentation with paging - instead of mapping the entire address space with the page table, which will be sparse usually, instead create page tables for the stack, heap, code, segments. Each process will have three page tables. Two bits at the start of the virtual address used to denote segment so on a TLB miss the correct page table can but used. In this case - Base and bounds in the MMU don’t refer to base physical address of segment and limit, but now they refer to the physical address of the page table for that segment and its limit (how many pages it has)
	- Way less space used for page table for most processes
	- Reintroducing segmentation issues (fragmentation) - need to store / fit the page tables in memory 
	- Program with a large heap etc still going to have a big page table 
- Another approach - **Multilevel page table** - avoid all the invalid regions, try not to introduce frag. Split the page table into pages and introduce a **Page Directory** to track where each page of the page table is, or if that has no valid data (so doesn’t need to be represented in the page table). Virtual address top bits index into the page directory, then into page table and then offset
	- Way more space efficient 
	- Used in modern systems 
	- Space time tradeoff - less space but another layer of indirection so if there’s a TLB miss another read is needed to find the page in the page table, and then use the entry to translate the address 
- More levels can be added - not enough, same idea though - page directory then split into pages, managed by another page directory
- Another approach - **inverted page table** - one single page table that represents pages of physical memory, entries contain info about which processs is using it 

Beyond physical memory: mechanisms
-  Whole processes (or multiple processes) can’t fit in physical memory at once
- Want to support but stay behind the seems to the programmer 
- Introduce a larger slower storage device (hard drive, ssd etc) to use to extend memory 
- Save some space on the disk to use for swapping out pages **swap space**
	- Swap space size ultimately determines the number of pages that can be in use by the system at a time 
- OS needs to remember **disk address** of pages
- A **present bit** is added to the PTE to represent whether the page is in physical memory or not. If it is not, it raises a **page fault** and runs a page fault handler 
- When the page isn’t in physical memory, address in the pte is disk location - handler looks it up, disk io to fetch, swap into physical memory and update the PTE with location in physical memory (page frame number)
- Blocking on I/O convenient time for processes to run
- If no space in physical memory, must swap out pages - **page replacement policy**
- OS doesn’t wait for the disk to be full - **high watermark** and **low watermark** - when there are fewer than low watermark free pages in physical memory, a **page daemon** runs and swaps out pages until there are spots for high watermark pages 
- Clustering or grouping pages to swap is more efficient 
- Daemon is based maxwells demon! (Ref on page 255)

Beyond physical memory: policies (replacement)
- When there is memory pressure (not a lot of free memory) - and page faults, OS must decide which pages to evict from memory to swap the pages it needs in from disk.
- Looking for optimal **replacement policy** to minimize misses / page faults
- Optimal strategy (can’t be implemented practically) - used to compare to other algorithms- evict the page that will be used furthest in the future 
- Simple Strategy: FIFO - simple but performance not good 
- Simple Strategy: Random - simple but, well, random - sometimes as good as optimal sometimes much worse 
- LRU - using history - base it on locality, either using frequency or recency to rank pages, replacing the least 
- Workload examples - LRU is generally better when the workload has locality 
- Implementing LRU - it would be expensive to find the actual least recently used page, instead it’s approximated 
	- A **use bit** is set to one whenever a page is used (can be in the page table or array) by the hardware 
	- OS can employ the use bit to approximate LRU in different ways - one way is the **clock algorithm**
		- Point at some particular page to begin with, when a replacement must happen, checks the use bit of the page it’s pointed at. If it’s 0 replace, if it’s one, set to zero and move to the next 
		- Dirty pages (modified) are more expensive to evict due to the disk io, a common modification is to prefer clean and unused pages to replace first, then dirty but unused 
- Bringing pages in to memory - **page selection**
	- Demand paging - bring it in when accessed 
	- Prefetching - bring in when it’s likely to be used 
- Thrashing - continuously paging 
- Admission control - when memory is tight OS might decide not to run a subset of processes to let the existing processes run better 
	- “Sometimes better to do less work well than to try to do everything at once poorly”

Building a complete virtual memory system
Extra things that should be consider when everything is put together
- VAX system - 70’s but many ideas still used 
	- No reference bit in page table to determine which pages in use - make all as inaccessible, track which ones trap - those are being used currently, put back their normal permissions 
	- They were worried about “memory hogs” - processes that use a lot of memory so physical memory with pages they need, introduced **segmented FIFO** instead - each process has a FIFO list and limit on how many pages it can keep in memory, evicts oldest 
		- Performance not good: **second chance** lists - when evicted the page moved to a secondary list (one dirty one clean) - if a page needed by another process, can remove one from clean list and load the page it needs. If the original process needs it again before that happens, it reclaims it from the free / dirt lost - so saving what would have been disk io
	- **Demand zeroing** - need to zero memory out so a process can’t read what was there before - instead of doing it on alloc, OS will basically wait and zero out what is needed on demand
	- **Copy-on-write** - whenever the OS needs to copy a page from one address space to another, it maps it into the new processes address space and marks them both as read only, if never written, it never actually needs to copy 
- Linux VM system - still going of course! 
	- similar virtual address space split - new thing is logical vs virtual kernel memory - logical section of kernel memory is contiguous and maps to physical memory directly, which is useful for **direct memory access** (fam) for device integration 
	- multilevel page table and supports many page sizes 
	- page cache - uses **2Q** replacement instead of normal LRU to handle the case with a cyclic large file is read (would wipe out the whole cache normally) - inactive list and active list - pages first go into the inactive list, if they’re used again they going into the active list - replacement takes from the inactive list. Similar performance to LRU but handles the worst case better. 
	- Security
		- Buffer overflow exploits 
		- **Return oriented programming** —> address space randomization makes this very difficult 
		- **Meltdown and Spectre** - showed that CPU speculative execution could be exploited. Move most of kernel address space out of process address space )stopped mapping it in) expect for the bare minimum - but then requires page table switch so it’s a performance hit in exchange for security

## Part 2: Concurrency

Introduction + interlude 
- multithreading - sharing address space - need to sync 
- we need synchronization primatives to avoid race conditions, etc - how does the hardware support that? 
	- thread api review 
Locks 
- How to build a lock? Need some support from the hardware, can achieve in different ways…
	- Lock metrics: mutex, performance, fairness
- Early versions in single threaded cpus disabled interrupts - can be abused, doesn’t work on multithreaded cpus, causes problems for missed interrupts - OS can still use it for its own purposes in some cases 
- Simple flag bar and spin waiting doesn’t work and is not performant 
- With some extra hardware support we can get it working - **test-and-set** aka **atomic exchange**, load/store unsigned byte (ldstub), locked atomic exchange (xchg) - depends on architecture - sets to the new value and returns what the old was, but it’s all atomic 
- Petersons algorithm- aside about history - there was a time when researchers were looking for ways to sync without hardware support, not relevant anymore but still cool 
- Simple **spin-lock** becomes possible with rest-and-set primitives - still not efficient but simple, no fairness guarantee 
- Another hardware primitive is **compare-and-swap** (or **compare-and-exchange**) - if the old value is what you expected, set to new value, otherwise do nothing - both return old value 
- **Load-linked / store conditional** - load the value into register with load linked, try to set it with store conditional - if no other thread has already set with store conditional, it sets and succeeds, otherwise do nothing and fails 
- Another one is **fetch-and-add** - atomically increment a value and return the old value 
- **Ticket-lock** - cool idea, uses fetch and add to assign a ticket to the thread that wants to get the lock, each time it’s unlocked, the turn variable is incremented and threads only get the lock when the turn value is equal to their ticket
	- First example that guarantees all waiting threads get a turn 
- Yielding instead of spinning is better, but can still be improved 
- Avoiding spin locks and improving fairness — enqueue next thread to get lock in a queue whenever they try to get it and it’s locked, then they will run fairly, combine with yield and there’s also no spin locking 
- A little spinning and then sleeping if the lock isn’t acquired right away is called “two phase” lock, can be beneficial (no context switching from sleep)

Lock-based concurrent data structures 
- adding locks to data structures to make them thread safe 
- simple approach is just adding a single lock around data accesses, but might not be performant 
- Ex: counter - simple lock approach works but scales poorly to many threads - approximate counter solution with local and global counters
- Ex: linked list - single lock works but has scaling issues, can use a lock per node instead, **hand over hand** - next node lock is acquired before current nodes lock is released in list traversal etc - not faster just due to all the overhead in most situations - hybrid might be able to work
- Concurrent queue - could use one big lock, also can use a lock for head and tail, stick a dummy node in the middle to keep them separate 
- Concurrent hash table - build from concurrent list, end up with a list / lock per bucket of the hash table

Condition variables 
- can spin wait using locks but very inefficient 
- sleep thread and wake it up when a another thread signals that a condition is met - uses signal / wait or lang equivalent - always have the lock when calling wait or signal as a rule of thumb to avoid trouble 
- state variable to avoid race conditions 
- Ex: producer / consumer w/ sync demo - using two condition variables to make sure that the right type of thread is signaled (producer or consumer) on the right event 
- Some cases you need to wake all waiting threads - eg broadcast vs signal - example with allocator makes sense 

Semaphores
- one stop shop for sync introduced by dijkstra 
- single int val in there, post and wait - wait decrements the value by 1, if it’s negative, calling thread suspended and waits to be woken on post, post increments value by 1 and if there are waiting threads, wakes one 
- easily used in place of locks - binary semaphore (only two states) - init value important - “number of resources available” - eg for a lock it’s 1, thread will call wait, decrement, and keep going if no other thread called, then when it’s done increase back to 1 on post - if another calls in critical sections, it will drop to -1 and the caller will wait
- use for ordering - like a condition var - can init to 0, so wait will block (for instance in a main thread) until something is done and a post is called in a worker thread, for example 
- producer consumer- use a full and empty semaphore to sync - producer waits on empty and posts to full when it has added a value, consumer waits on full and posts to empty - need a regular lock still on data structure 
- reader/writer lock - many readers only one writer at a time - writers work like normal lock, only one can get the write lock at a time - keep a count of readers, get the write lock when first readers gets requests read lock, increase reader count - when last reader is done, release write lock. Cool semaphore example - easy to starve writers, more sophisticated examples exist but overhead can be high versus simple lock 
- Hill’s Law - “big and dumb is better” - in reference to his dissertation on how to design caches for CPUs
- Dining Philosophers - famous problem, practical utility is low - key was to change the order for one philosopher, they all try to get the left fork and then the right, except the last one, who tried to get the right then the left - this avoids the case where they all grab the left fork and deadlock 
	- Similar problems: cigarette smokers problem, sleeping barber problem 
- Semaphores can be used for throttling - just init to the max number of threads allowed in the region of interest and wrap in wait/post - value of one is back to our lock but higher value will only let X threads be in there at a time 
- Implementation of semaphores with locks and condition variables is pretty straightforward (page 407) - making condition variables out of semaphores is harder
- “Don’t generalize; generalizations are generally wrong” Lampson

Common concurrency bugs
- common bugs in real systems, based on a study of open source projects (MySQL, Apache, Mozilla, and OpenOffice)
- non-deadlock bugs - the majority fell into two categories
	- Atomicity violation - code that is assumed to atomic is not, basically missing a lock around some shared state, at least in the example given 
	- Order violation - missing sync where order matters, eg thread needs to wait for some resource to be available, order is assumed but not enforced with sync primitives - solve with a condition variable and lock or using semaphore
- deadlock - how can avoid, prevent, detect and recover from deadlock situations? 
	- interesting example from Java vector class and add all - wonder if that’s still an issue (could deadlock if run in two threads at the same time)
	- Needed for deadlock: mutex, no preemption, hold-and-wait, circular wait
	- Preventing deadlock
		- remove circular wait - Lock ordering - always get multiple locks in the same order to avoid deadlock, partial ordering possible in larger systems, example from Linux 
		- remove Hold-and-wait - put a big lock around the acquisition/ release of the other locks - then order won’t matter only one thread can be in there at a time - not great for performance / concurrency hit 
		- Remove “no preemption” (kind of) - use a try lock routine, instead of blocking returns on failure to get the lock, so you can unlock other locked resources and try again later - the programmer is sort of preempting their own ability to keep the lock - “live lock” is possible here but pretty unlikely, can add random delay to retry 
		- Remove “mutex” - design the data structure / system in a way that doesn’t need locking / waiting - using “compare and swap” atomic operations, etc - this is difficult in general 
		- Avoidance using scheduling - a smart scheduler could look at rhe locks needed by each thread and compute in which combinations a deadlock could occur, and then avoid those / schedule around them 
			- Not really practical except in specific situations / embedded 
			- Can hurt perf / reduces concurrency 
		- Detect and recover - some db systems have a deadlock detector that runs and checks for dependency cycles / deadlocks, will prompt or trigger restart etc to recover 

Event-based Concurrency 
- loop until and event occurs, when it does, run some handler 
- how can you check for events? select / poll etc - checks for file descriptors with incoming data to read and file descriptors that can accept a response (write) - server loops for ever and gets data when select / poll reruns descriptors with data available 
- watch out for blocking the event loop, blocking io calls - some systems provide a async io interfaces, file reads etc
- introducing threading back in can be tricky, now sync primitives are needed again 

## Part 3: persistence 
**I/O devices intro** - how to get data in and out 
Hierarchical - CPU connected to fast bus with memory and graphics connections, slow busses for disk, mouse keyboard etc
Hardware devices need to expose API for control - common format is status buffer, command buffer, data buffer - poll status for device not being busy, write the data, write the command, continue polling status until the device is done 
- PIO - programmed io - if the main thread is doing the data writing
- can we avoid spinning when polling? Yes - using interrupts… calling process / thread starts call to device and instead of polling can yield so that the cpu can do other things, when it finishes an interrupt will run and the cpu can switch back to caller and continue
- interrupts not always better - if the device is fast, the context switch and interrupt handler is slower than polling a bit - if it’s slow it’s better to use interrupts - can use a hybrid approach if you don’t know, try to pool a bit and if it’s not done use an interrupt 
- they can also overwhelm the system and lead to livelock - where the system is only running interrupt handlers - can be coalesced, eg wait a little on a interrupt and if you get more, handle them together - but it’s a tradeoff that can add to latency 
- getting info to the device - instead of using the CPU to copy, can use specific device - **Direct Memory Access** engine - handles the copy of memory to the device, CPU can switch instead of wasting cycles copying 
- Actual interface to device - can be explicit I/O instructions (in/out to read write registers on device) or use **memory mapped I/O** - device registers available as if they were memory locations and hardware handles the routing and normal load / store instructions can be used by OS - both methods used 
- Device drivers wrap the device specifics and expose the public api to the OS so it doesn’t need to know / care about the device it’s using 

Hard disk drives 
- overview of hdd - how they work etc, useful to understand 
- interface - drives split into 512 byte sectors that can be read or written - multi sector write is fine but only single sector write is guaranteed to be atomic 
- spinning disk with concentric circles (tracks) and disk arm to read - seek - moving to read a sector on a another track slow - rotational delay also waiting for the data to be under the arm 
- total time to read the disk - time to seek (arm to correct track), time to rotate (so requested data is under head), time to transfer (actually reading the data)
- average seek is 1/3 of full seek ( given by manufacturers usually ) page 467 has a derivation
- given high cost of disk io - **disk scheduling** becomes very important - disk scheduler decides how to organize access 
	- unlike scheduling processes, the disk scheduler can get a good estimate of how long each will take
	- It can use something like **shortest job first** but need to avoid starvation
	- **shortest seek time first** - but os doesn’t know geometry to it’s really nearest block (sector) first - still has a starvation issue if there’s a bunch of reads / writes for blocks near each other, accesses for further blocks won’t be fairly served 
	- **Elevator (SCAN)** - instead of a going to the next closest sector / track - the arm sweeps back and fourth across the disk - incoming requests during a sweep can either be queues for the next sweep if the arm has already passed that track this sweep. Like an elevator! 
		- Still doesn’t account for rotation though 
	- **Shortest positioning time first** - also shortest access time first - when seek time and rotation time are roughly equivalent (modern drives atm) it makes sense to consider both in deciding which to service next - OS doesn’t have all this info though so it can’t schedule, needs to be handled by the drive 

RAID - redundant array of inexpensive disks
- put a bunch of drives together into an array and manage by separate dedicated system, microcontroller or CPU
- appears to client as a big drive, just like any other drive, easy to switch to
- goal is to increase reliability, performance and / or capacity- different RAID levels trade these  off differently
- to consider performance - very dependent on specific workload - random (most time spent seeking and waiting for spin) vs sequential (most time spent transferring data) - they are considered separately and random usually much much slower 
- Level 0 (striping)
	- No actual redundancy- spread sequential blocks across the array - so sequential reads / writes can be done in parallel 
	- best performance, best capacity (sum of disk capacities), no reliability (any single disk failure results in data loss) - throughput is just single disk throughout times number of disks
- Level 1 (mirroring) - what it sounds like - half the capacity because disks are Literally mirrored, more reliable in case a disk fails, writes need to happen to mirrored disks now - half the sequential read and write throughput - random reads actually pretty good, throughput is max - writes still half max (one logical write is two physical)
- Level 4/5 (parity based redundancy) - uses one disk to store parity information (xor across a row of sectors) - if any sctor in a row is lost it can be reconstructed. Smaller capacity (reduced by 1 - the disk used for parity info). Reliability- can handle one disk failure. Sequential read and write - (n-1) x disk value - the parity disk becomes a bottleneck, “small write problem” - even though writes can happen in parallel, they’re serialized because they all need to update the parity disk, so random write performance is bad - half the disk value (half because each update is a read and a write), doesn’t scale with more disks 
- RAID 5 is just like 4 but the parity data for a stripe is rotated through all the disks - better random write throughput because it can be parallelized and now scales with number of disks - basically has completely replaced level 4 except in very specific cases where the simplicity is important and / or workload is such that they don’t care about the small write problem 
- Page 489 has a good table with throughputs of all and how they scale 
- Consistent Update Problem - need to update multiple disks ideally atomically, in case of failure, in practice something like a WAL is used to record what is about to happen, and can be used to replay / recover

Interlude: files and directories 
We’ve seen the abstraction for the CPU (processes) and for memory (address space) - how do we manage a persistent disk? (spoiler: files and directories)
- file - simple linear array of bytes - internal number and name (**inode number**)
- directories are lists of (low level name “inode”, user readable name) pairs - and they can be nested of course to create hierarchies 
- file api
	- open - returns a file descriptor - process specific, so some type of array of handles to open files is stored in the proc structure (unix based)
	- read / write - sequential 
	- Non sequential read/write - lseek
- file table entries are usually independent- but they can be shared in some cases
	- when child is created with fork
	- Using dup system call - creates a new file descriptor that refers to the Sam’s underlying open file 
	- reference count is used - not removed until all references gone 
- writes are buffered - fsync (or non Unix equivalent) can be used to forced writes to disk if needed - if a file is newly created, it can also be necessary to fsync the directory that contains it 
- rename - needs to be atomic wrt to crashes - other things can be built on it 
- meta data stored in inode structure- access with stat or fstat / similar 
- unlink decreases ref count of actual inode  - removes a inode to human readable name in structure 
- hard links - simple additional “links” between the same inode and human readable name in the file data structure
	- Cool but limited - can’t be used for directories, must be on the same partition (inode numbers only unique to partition)
- symbolic links - looks similar but implemented differently - it’s actually a file of type “link” that points to the file of interest, can have a “dangling reference” if the file pointed to is deleted 
- create empty file system mkfs or similar - and mount any partition to any did that you want with mount to make it appear as all one - eg mount /dev/sda1 /home/user - you can see all these running mount with no args

Implementing a filesystem
A simple file system similar to Unix but simpler - cool thing is that it’s all software, so with that flexibility many different file systems have been developed
- Organization of the Very Simple File System (VSFS)
	- Blocks of let’s say 4K 
	- Most of them are reserved for user data - **data region**
	- Some used for **inodes** - structures with information about files
	- An **inode bitmap** and **data bitmap** - to show the free / used blocks
	- Super block - metadata about the number of inodes, size of data region, location of bitmaps, etc - first entry contains indication of file system type 
	- Inode data structure contains all the file metadata and pointers to location on disk 
	- If the inode stores direct pointers to blocks on disk for the file data - size is limited to the block size times number of pointers
		- Multilevel indexing - store direct pointers —> most files are small
		- But also indirect - a pointer to block that contains a list of pointers to data - supports bigger files
		- Can add levels of indirection to support even larger files - pointer in inode points to list of pointers that points to list data pointers, etc
	- Directory structure - basically on disk structure looks like a list of the inode number and name (with lengths) of all the files that are “in” the directory - and of course the . and .. entries. its actually represented usually with its own inode in the inode table and special type of file (directory) instead of regular file. Unlinking a file can leave an empty space in the directory- that’s why the lengths (of the record and name) are stored with the files so the space be reused. 
- Access paths - actually reading / writing a file 
	- Starting with mounted file system - super block in memory - everything else on disk, what happens? Say opening /foo/bar
	- start at “well known” inode for root - once root inode is read in, data pointers read in, checked for “foo” 
	- Once foo is found, it has the inode for it too, start reading its data pointers to look for bar 
	- Once it finds bar - read its inode into memory, allocate a file descriptor and return to the user 
	- Reads can be called on the file descriptor to read blocks from the file - the file table will store the current offset for read etc for each subsequent read 
	- Close just deallocates the file descriptor 
	- Figure 40.3
	- Writes require more IO - can be 5 I/Os in this example, and creates are even worse - 10 I/Os 
	- Create
		- Read root inode
		- Read root data (looking for foo)
		- Read foo inode
		- Read foo data looking for bar 
		- Read inode bitmap (for free space)
		- Write inode bitmap (to mark space used )
		- Write foo data
		- Read bar inode
		- Write bar inode (add foo)
		- Write foo inode
	- Write (already open)
		- Read bar inode
		- Read data bitmap
		- Write data bitmap
		- Write bar data 
		- Write bar inode
- How to optimize so that all this I/O is efficient enough to use? Spoiler: caching / buffering! 
- Old systems had a dedicated cache for disk I/O to cache important blocks - better to be dynamic so RAM can be used for other things if you don’t need that much set aside for the cache - **unified page cache**
- Writes buffered for up to 1-30 seconds so they can be batched and improve performance- if system dies buffered writes are lost, databases handle with forcing writes (fsync, etc) or WAL
- Summary - basic machinery for a file system
	- Need information about file metadata (inode)
	- Directories are special file type that store name inode mappings
	- Other structures: bitmaps to store free or allocated inodes and data blocks 

Fast file system
Original Unix file system we as in the example on disk: 
- super block, inodes, data 
- Super block with all the needed pointers, sizes etc

This was simple to reason about but performance is very bad and gets worse as it runs. Files become fragmented over time because of his free space is managed. Also reading an inode and then going to the data (which happens all the time) causes a seek. Basically - it treated it like memory and not a disk, and performance was affected. 

Crux: how do you organize on disk data to improve performance? 

**Fast file system** developed by group at Berkeley 
- keep same interface but change implementation to optimize for hardware
- Disk awareness 
- Introduces **cylinder groups** or **disk groups** - a group of all the data that’s the same distance from the center of the drive —> doesn’t need an expensive seek to move within a group
- all the data needed to manage the group independently is in the group, super block, inode / data bitmap, inodes, and data 
- Keep related things close (and unrelated things far away)
	- Put directories in the group with not many directories and more free space (to add files)
	- Keeps file data in the group their inode is in (no more seek between this super common sequence) 
	- Create files in the in the same group as their directory, if possible
- Large files: instead of letting large files fill the whole group, after a certain chunk size worth of blocks, their data is spread to other groups in chunks. This makes it possible to put other files in the original group and still benefit from locality. There is still a seek cost to read the data in other groups sequentially, but if the chunk is big enough it’s amortized by the the data read/write
- Summary: treat a disk like it’s a disk - improvements in FFS inspired modern Linux file system and many others 

Crash Consistency: fsck and journaling (wal)
The system could lose power or crash at any time, maintaining the file system offense requires multiple writes - need to deal with or avoid inconsistent states if the system crashes between two operations. Original way of dealing with it was using **fsck** - now it’s more common to use **journaling (write ahead logging).**

fsck - look for inconsistency after the fact and clean them up, would run before a disk was mounted - required intimate knowledge of the file system to be able to check everything. Biggest issue though was slowness - scanning the whole disk to check for inconsistencies, build the real data bitmap to compare to the disk one, etc, takes a while as disks get larger. Needed another option…

journaling / WAL - before doing any operation write them to a log in a different location - more work for updates / writes but much less work to recover (you know exactly what to check). Same idea used in databases. Rough idea of how it works: 
- Write transaction to log (separate partition maybe even separate device)
- It has: tax start marker, blocks from, blocks to, tx end marker - physical logging, eg it literally has all the data the data we’re using. 
- Once log is written system is “checkpointed” up by carrying out the transaction - if it completes successfully it’s up to date. 
What happens if the system crashes while writing the WAL entry though? Need a way to: make sure that no bad transaction can get written to the log (and look good) - but also don’t want it to be too slow. 
Steps: 
- **journal write** - write everything for the current journal entry except for the transaction end, wait for the writes to complete
- **journal commit** - once the write is confirmed, then write a single 512 byte block transaction end, wait for it to complete - write is “committed” 
- **checkpoint** - actually carry out the transaction 
- **free** - update (journal superblock) to track the lost recent and oldest uncommitted tx (or none of all complete) so the space can be used again
The blocking part of this (waiting for the entry to be written before committing) can be fixed by introducing a checksum and allowing the log entry to write all at the same time as a usual write - this is used in ext4

Recovery - on startup, replay any committed transactions that haven’t been applied - disk will be in a consistent state and can be mounted. 

Full data journaling - we now have a way to recover but must write the data twice for each write (writing to the log and then to the location on disk). Another approach - **metadata journaling** - write everything to the log except the actual data blocks, write them to their home. Only difference- need to wait for the data blocks and the journal entry to be complete before ”committing” by writing the transaction end block.

Windows NTFS, SGI’s XFS both use metadata journaling - ext3 has data journaling or metadata journaling modes available as an option. 

Summary: fsck too slow to actually recover, ordered metadata journaling is the most common way to provide reasonable integrity but without requiring the extra write that data journaling does. 

Other ideas exist - log centered file system, adding back pointers to data-blocks so that it’s easier to check consistency (point back to the inode it belongs to, etc), “soft updates” - order writes in way that nothing is ever inconsistent- really difficult to implement because it needs to know a lot about the actual file system. Finally **copy-on-write** - not actually updating any of the structures, instead write somewhere else, and when all completed, update the pointers etc. 


**Log Structured File Systems**
A file system developed at Berkeley- based on observations: 
- as memory increases, more data can fit in cache - so performance is mostly determined by write performance 
- sequential I/O on disk way more performant than random (seek and rotation delays)
- existing file systems were performing poorly on common workloads - create a small file, for ex - with FFS is updating inode bitmap, data bitmap, creating the inode, creating the data block, update directory that holds the file 
- not raid aware
LFS buffers all writes in an in memory segment - and when it’s full writes the whole segment sequentially.
- how much to buffer? Depends on the trade between transfer time and positioning time - each individual write will have a cost to get to the right position - transfer time is constant - so need to group enough together that positioning doesn’t dominate 
How do you find inodes? (They’re just dumped sequentially with the data blocks)
- introduce an imap - map to where the most recent version of an inode is on the disk, but the imap itself is also spread out and dumped with the data and inode info in the segment
- add checkpoint region, that points to the locations of imap chunks - in a fixed location 

Directories are basically the same as classic structure (inode, name mapping)
- using imap to track locations of inodes helps - if it was the location, when a file was updated, its location would change (no in place overwriting in LFS) - the directory that pointed to it would need to update too, and so on (**recursive update problem**)

Garbage collection
- in this system - each update writes the inode and updated data again - leaving the old versions of the data on disk but not “live” 
- can actually keep - **versioning file system** - read about later 
- LFS reads a bunch of partially used segments, determines what is still live and what is garbage, and then cleans up / compacts into fewer segments - freeing up space for more writing
- whether a block is alive? 
	- add a **segment summary** block to the segment - which file each block in the segment belongs to and which in the file it is 
	- can use with imap - look up block address for inode in imap, adjust for offset - if it’s the current block in question it’s live - otherwise it can be removed 
- When to clean? - periodically, when you have to, or when idle
- What to clean? What is worth cleaning? - more challenging
	- Originally the split into “cold” and “hot” segments - hot had files that were updated a lot - can wait to clean, cold we’re pretty stable - clean sooner 

Crash recovery
- introduce a log 
- Checkpoint region points to head and tail segment, each segment points at next segment to be written 
	- Crashes during writing to checkpoint region - handled by putting two, one at each end of the disk - start updates with header and timestamp, write body, end with block with timestamp - inconsistency from a crash can be detected and the CR that is latest and consistent can be used 
	- roll forward on - starts with last checkpoint region, read through log and apply updates that found to get as recovered as possible 

Summary
LFS introduces copy-on-write or shadow paging - no in place overwrite. Main advantage is large writes - way more performant. Generates garbage though, old copies of files spread all over - needs cleaning. Cleaning costs the main controversy with LFS, but ideas inspired many modern file systems. 

Flash-based SSDs
No moving parts - but to write a flash page must delete a flash block, and they eventually wear out.

Crux: how do you use them but deal with those two issues? 

Bits are represented by the charge on a transistor - **single level cells** (SLC) represent one bit (high for 1 low for 0), there are also **multi-level cells** that represent 2 or 3 bits per transistor. SLC are more expensive and higher performance. 

Cells organized into blocks (erase blocks) of 128kb or 256kb (usually), and pages ~4kb - obviously different from blocks and pages in other contexts. 

The interesting thing about flash SSDs is to write to a single page in a block, you must erase the whole block. 

Ops:
- Read (page) - can read anywhere, any page, time doesn’t depend on previous read location - random access device, fast (microseconds )
- Erase (block) - must erase the whole block before programming any page in it, sets everything to 1, kinds of slow operation (milliseconds)
- Program (page) - once a block is erased, program can be used to change some of the 1s to 0s in a page - faster than erase but slower than read, 100s microseconds 

The pages basically have state: 
- invalid (initial)
- erased - erased (all ones)
- valid - already programmed 

The only way to write to a valid page, is as above, erase the whole block of pages. 

Raw flash to an actual flash based SSD
- build the SSD out of a bunch of flash chips, behind a controller with its own volatile memory 
- **Flash translation layer** (FTL) adapter to convert os read / write requests to flash - logic blocks, etc to physical 

Performance
- multiple flash chips used in parallel
- design to avoid **write amplification**

Reliability 
- main concern is wear out - so **wear leveling** - try to use all blocks the same so they wear at the same rate 
- minimize disturbance- program pages in order to minimize disturbance 

FTL organization 
- bad approach - directly map logical to physical - results in wear out depending on workload and really poor write performance (worse than platter)
- use log structured system like with file systems
	- always write to the next free page (spreads over the whole disk)
	- needs a mapping table - logical address to physical address
	- main issue: garbage collection (with no overwrite there’s a lot of old versions around that can be cleaned up)
	- main issue: size of mapping table 
	- garbage collection: checks for physical pages not pointed to by the map, read live pages in the block, erase block, and write out the live pages from erased block
	- mapping table size: for a 1Tb SSD with one entry for 4kb page (4 byte entry), would be 1 Gb of memory
		- could use block based mapping instead - map logical address to a physical block and offset - issue is with small writes, potentially need to read in whole block, it back to a new spot with the new data - and created a garbage block
	- Solution: hybrid approach to mapping - keep some empty “log blocks” available with full page mapping “log map”, and the rest of blocks in a “data map”. Need to merge and “full merge” to move pages together and keep log blocks so ita a little complex
	- Solution: a simpler option - cache recently used translations - can perform poorly though depending on workload
- The FTL also has to do **wear leveling** - log system helps but sometimes data is long lived, so even though the log system spreads it out, some blocks can wear at different rates. To compensate it reads data in from those blocks and writes elsewhere so they wear evenly with the rest of the drive

Data integrity / protection 

Crux: how do you make sure you get the same data out that you put in? 

If the whole disk fails - it’s either working or not and easy to detect, RAID saves you - but other failure modes are possible, where it looks like the disk is working but it has issues with accessing certain sections 
- Latent sector errors
- Block corruption 

Latent sector errors - caused by damage to disk in a sector, could be head crash, and cosmic rays can flip bits - error correcting codes can be used to check the data is as expected and recover it if possible, but if it can’t, the disk returns an error. 

Block corruption - more sinister, not detected by the disk, faulty bus or firmware, etc something causes the bits that are written to not be what was expected but the disk can’t tell 

Handling the partial failures:
- LSE easy to detect - when found, try to use whatever redundancy mechanism is in place to restore the data (RAID)
- Corruption - detect using a checksum and restore in the same way as LSE once you find it (the hard part is finding it)

Other common failure modes: 
- Misdirected writes - the controller writes the right data to the wrong block, or even the wrong disk
	- Adding physical id for disk location and block location to the checksum makes this detectable (otherwise the checksum would work)
- Lost writes - the write is reported as completed in upper abstraction layers but didn’t actually complete - this isn’t handled by the checksums above. Some systems read back the data they wrote to verify but that doubles io per write. ZFS adds checksums to inodes and indirect blocks so that if the write to block is lost, the checksum with the old data will fail and it can be detected 

Scrubbing - run through all the data, check the checksums and fix errors, usually on a schedule (daily, weekly, etc)

**Distribution**

Distributed systems
Communication - it’s unreliable, how do we deal that? Packet loss is inevitable, etc

Unreliable communication - sometimes ok, eg UDP - sender has know information about whether the datagram was received, it might not have been, etc

Reliable communication - send ack, use id that increments to make sure duplicate messages don’t get sent down to receiving program - TCP

Communication abstractions
- people tried to apply OS abstractions to build shared memory systems, didn’t work though because page faults are costly if you have to go to another machine and dealing with failure is hard (a whole chunk of the address space could disappear if a machine goes down)
- RPC - more usable - send request with the data and function to call on a remote machine, run the function there and return the results to the client 


**Sun’s Network File System (NFS)**
Distributed file system - Sun was one of the first, and they opted for an open protocol, NFS servers still around

**Andrews File System (AFS)**
- Developed at Carnegie Mellon University in the 80s
- Trying to address some of the issues with NFS
- New NFS type systems use ideas from AFS
