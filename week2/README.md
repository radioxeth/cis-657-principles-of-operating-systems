# Week 2 Lists and Queues

## Directory
- [Home](/README.md#table-of-contents)
- [Week 1 Introduction and Review, Concurrency, and Shared Memory](/week1/README.md#week-1-introduction-and-review-concurrency-and-shared-memory)
- **&rarr;[Week 2 Lists and Queues](/week2/README.md#week-2-lists-and-queues)**
- [Week 3 CPU Scheduling](/week3/README.md#week-3-cpu-scheduling)

## 2.2 Interrupts, Traps, and System Calls
([top](#directory))

### Entry into the Kernel
- Methods of entering the Kernel 

- hardware interrupt
  - i/o device (disk, network)
  - clock (used for scheduling, time of day)
- hardware trap
  - divide by 0, illegal memory reference
- software-initiate trap
  - system call: service provided to non-kernel process

**all follow the same basic steps**

### Entery into the kernel II

- first, kernel must save machine state
  - why?
  - think about how instructions are executed on a CPU
  - what context information is used, an dhow might it be lost?


### Why save machine state?
- We don't know what code we're switching to
- instructions modify data registers
- PC points to new instruction
- PSW can be changed
- other process-specific state we haven't talked about yet

### Responsibilities of OS and HW
- Save entire state of processor, including PC and status registers
- Run appropriate interrupt hander routine.
  - already in memory before interrupt
  - typically loaded at boot or device connection time
- restore *entire* processor state and continue on a point of interruption

### Example Kernel Entry Sequence
- hardware switches to kernel mode
- Hardware pushes PC, PSW, trap info onto stack
  - what is saved here is defined by ISA
  - some OS use a separate interrupt stack to avoid corruption
- additional assembler routine saves all other state that the hardware doesn't
- kernel calls a C routine - the handler

### Entery into the Kernel IV
- handlers for each kind of entry:
  - system calls
  - hardware traps
  - interrupt handlers for devices
- each kind of handler takes specific parameters (eg syscall number, exception time, or the unit number for an interrupt)

### Handling an Interrupt
- process 0 is running add instruction

proc 0
```
lw $v3, varx
add $v3, 5  <--PC
sw $v3, varx
```

- a packet arrives at the network interface
- interface posts interrupt

- after interrupted instruction completes:
  - all in microcode
  - No explicit instruction execution
  - push registers onto stack
  - read interrupt number from bus
  - look up interrup number in interrupt table (or vector)
  - branch to location in vector

### Steps 1 and 2

interrupted instruction completes
proc 0
```
lw $v3, varx
add $v3, 5  
             <--PC
sw $v3, varx
```
State saved on stack
|stack||
|-|-|
||&larr; old SP|
|saved R0||
|saved R1||
|saved R2||
|...|...|
|saved R13 (SP)||
|saved R14||
|saved R15 (PC)||
|saved PSW||
||&larr; new SP|

### Steps 3 and 4

|stack|interrupt number|
|-|-|
|`serial_handler`|1|
|`wifi_handler`|2 &larr;|
|`SATA_handler`|3|

Interrupt number 2&#8628;
```c
wifi_handler(){
    ...
}
```

### After Handler Finishes
- reverse the process on the way out
- restore state from stack
- restores PSW (PSR) and process mode
- resume at intstruction following interrupt
- process doesn't realize interrup happened

proc 0
```
lw $v3, varx
add $v3, 5  
sw $v3, varx <--PC
```

### Summary 
- interrupts, traps, and system calls use similar mechanisms
- on interrupt, HW automatically saves some state; OS must save the rest
- interrupt number used as index to interrupt vector
- handler called
- state restored and process resumed after handling interrupt

## 2.3 Fundamental OS Data Structures
([top](#directory))

### Lists and Queues
- Two fundamental data structures in OS
- lists of free memory regions
- queues of resource requests
- timer queues
- queues of prcoesses
  - ready queue
  - processes waiting on a resource

## Example in Xinu: process Lists
- doubly ended (explicit head, tail nodes)
- doubly linked (prev, next pointers)
- sequential access
  - only one process manipulates list at a time
  - enforced through locking
- single data structure used throughout the OS
- other OS have multiple types of process lists/queues (for example FreeBSD)

## Process Lists II
- Node stores two items
  - key
  - Process ID
- Either FIFO or sorted by descending Key
- A list node:

|Previous|PID|Key|Next|
|-|-|-|-|
|&larr;|-|-|&rarr;|

### Sample List
<img src='/week2/images/sampleList.png' width=500/>

- keys in descending order
- PID in each entry
- HEAD->prev and TAIL->next are both NULL

### Empty List

<img src='/week2/images/emptyList.png' width=500/>

- empy list has just head/tail nodes
- trade-off of space time
  - simplifies checking for empty list
  - removes source of errors


### Trade-offs

- space/time trade-off in lists is an example of fundamental pattern in OS design
- rarely is there one optimal choice
- often deciding to emphasize on characterstic over another
  - space vs time
  - different choices at different points of design
  - system features influence choice (eg large vs small memoery, target environment)

### Example of Influence of System Features
- dumb phones had dedicated, simple OS
- smartphones required new features
- european industry standardization on a new smartphone OS (symbian)
- us industry adapted two desktop OSs
   - both run the same browsers
   - iPhone 3GS = cray X-MP (super computer form 1960s)


## 2.4 Xinu queue.h
([top](#directory))

### Constants and Data Structures


- xinu has a fixed array length defined at compile time

```c
/* default # of queue entries: 1 per process plus 2 for ready
 list plus 2 for sleep list plus 2 per semaphore */
#ifndef NQENT
#define NQENT (NPROC+4+NSEM+NSEM)
#endif

#define EMPTY (-1)       /* null value fo qnext or qprev index */
#define MAXKEY 0x7FFFFFF /* max key that can be stored in queue */
#define MINKEY 0x8000000 /* min key that can be stored in queue */

struct query { // one per process plus two per list
  int32 qkey;  // key on which  the queue is ordered
  qid16 qnext; // index of next precess or tail
  qid16 qprev; // index of previous process or head
};

extern struct qentry queuetab[];
```

### inline queue manipulation

- processId is implicit
  - processId is the same as the entry in the queue table

```c
// inline q manipulation functions

#define queuehead(q) (q)
#define queuetail(q) (q)
#define firstid(q)   (queuetab[queuehead(q)].qnext)
#define lastid(q)    (queuetab[queuehead(q)].qprev)
#define isempty(q)   (firstid(q) >= NPROC)
#define nonempty(q)  (firstid(q) <  NPROC)
#define firstkey(q)  (queuetab[firstid(q)].qkey)
#define lastkey(q)   (queuetab[ lastid(q)].qkey)

/* inline to check queue id assumes interrups are disabled */

#define isbadqid(x)  (((int32)(x) < 0) || (int32)(x) >= NQENt-1)
```

### Key Points
- The inline queue manipulation functions are extremely efficient
- because there are no null pointers, queues will always have first and last items (the first item might be the tail, and the last item the head, they'll be there!)
- This justifies the trade-off of additional space used in the queue table

## 2.5 List Implementation 
([top](#directory))

### Didn't we already cover this?
- the prior unit described abstract or conceptual lists in Xinu
- The concrete implementation differs in two ways
  - relative pointers
  - implicit data structure
- This optimization is an example of a trade-off
- xinu is for embedded systems (internet of things)

### Relative Pointers
- Embedded systems generally have a small number of processes
- NPROC in Xinu typically < 100 (default is 8)
- PID range from 0 to NPROC-1
- use PID as array index
- desktop OS use much larger PID space
- trade-off: reuse vs indexing time

### Implicit Data Structure
- Key observation: A process (in Xinu) appears on only one list at a time
- Obviates need for PID in list node
- Use array implementation
- ${i^{th}}$ element of the array is for PID *i*
- Inserting node 3 in the list == putin PID 3 in the list

### Array Implementation
key in descending order

|idx|key|prev|next|
|-|-|-|-|
|0||||
|1||||
|2|14|4|61|
|3||||
|4|25|60|2|
|...||||
|NRPOC-1||||
|...||||
|60 (HEAD)|MAXKEY|-|4|
|61 (TAIL)|MINKEY|2|-|

- each row corresponds to 1 process
- each pair of rows is head/tail of 1 list
<img src='/week2/images/arrayImplementation.png' width=500>

## 2.6 FreeBSD Lists and Queues
([top](#directory))

### An Alternative Approach to Queues
- Traditional pointer-based lists
- Singly linked lists
- Doubly linked lists
- Singly linked tail queues
- Doubly linked tail queues

### Singly Linked Lists
- Single head pointer
- Only forward traversal (only forward pointers)
- Insert at head, or insert after existing item
- Memory efficient
- O(n) cost for iten removal
- Best used for 
  - LIFO queue (stack)
  - Datasets for few removals

### Singly Linked Tail Queue
- pointer to head and tail
- only forward traversal (no back pointers)
- Memory efficient
- O(n) cost for item removal
- Insertion options
  - Head of queue
  - After existing element
  - Tail of queue
- Best use: FIFO queue

### Lists
- Head pointer (no tail pointer)
- Doubly linked list
- 2x memory overhead
- O(1) removal cost
- Insertion
  - Head of list
  - before or after any existing item
- Still traversed only in forward direction

### Tail Queue
- (similar to Xinu implementation)
- Head and tail pointers
- Doubly linked lists
- 2x memory cost
- O(1) removal
- Insertion
  - Head of list
  - Before or after any existing item
  - tail of list
- can be traversed in either direction (finally!)

## 2.7 Basic Queue Extraction
([top](#directory))

(xinu queues)

### Three Extractoin Functions
- `getfirst()` remove and return head of the list
- `getlast()` remove and return tail of the list
- `getitem()` remove and return a specific process from a list (no searching required)

### `getfirst()`

```c
// getfirst - remove a proess from the front of a queue

pid31 getfirst(
  qid16  q       // id of queue from which to remove a process (assumed vali with no check)
){
  pid32 head;
  if(isempty(q)){
    return EMPTY;
  }
  head = queuehead(q);
  return getitem(queuetab[head].qnext);
}
```

### `getlast()`

```c
// getlast - remove a proess from the end of a queue

pid31 getlast(
  qid16  q       // id of queue from which to remove a process (assumed vali with no check)
){
  pid32 tail;
  if(isempty(q)){
    return EMPTY;
  }
  tail = queuetail(q);
  return getitem(queuetab[tail].qprev);
}
```

### `getitem()`

```c
// getitem - remove a proess from an arbitrary point in the queue

pid31 getitem(
  pid32  pid       // ID of process to remove
){
  pid32 prev, next;
  
  next = queuetab[pid].qnext; //following node in list
  prev = queuetab[pid].qprev; //previous node in list
  queuetab[prev].qnext = next;
  queuetab[next].qprev = prev;
  return pid;
}
```

- remove next and prev pointers from pid node at a higher level.

## 2.8 FIFO Queue Manipulation
([top](#directory))

### What is a FIFO Queue?
- first in first out
  - also called first come first served (FCFS)
- The basic queue type
  - Banks
  - Grocery stores
  - lines

### Two Routines
- `enqueue()`
  - insert item at the tail of the list
- `dequeue()`
  - remove and return item from head of the list
- neither is truly limited to FIFO queues
  - eg `dequeue()` can be used on any queues to return and remove the first item
  - uses `getfirst() and cleans up the the entry

### Starting Queue (one item)
<img src='/week2/images/oneItemQueue.png' width=500/>

- don't care what the key is, do not specify it

<img src='/week2/images/oneItemQueueConcrete.png' width=500/>

`enqueue(2,60)`

### `enqueue()`

```c
// enqueue - insert a process at the tail of a queue

pid32 enqueue(
  pid32    pid //process id to insert
  pid16    q   //queue id to use
)
{
  qid16 tail, prev; // tail and prev node indexes

  if(isbadqid(q)||isbadpid(pid)){
    return SYSERR;
  }

  tail = queuetail(q); //time saver, compute once
  prev = queuetab[tail].qprev; //queuetab[tail] always valid

  queuetab[pid].qnext = tail; //insert just before the tail node
  queuetab[pid].qprev = prev;
  queuetab[prev].qnext = pid;
  queuetab[tail].qprev = pid;

  // link in the new item just ahead of the tail.
  // note that prev could be head 
  // (if the queue were empty before)
  return pid;
}
```

### `dequeue()`

```c
//dequeue  -remove and return the first process on a list

pid32 dequeue(
  qid16        q //id queue to use
){
  pid32 pid; //id of process removed

  // check error
  if (isbadqid(q)){
    return SYSERR;
  } else if (isempty(q)){
    return EMPTY;
  }

  pid = getfirst(q);
  // have to clean up the pid node
  // clean up for security and errors
  // could point to information we don't want to reveal
  queuetab[pid].qprev = EMPTY;
  queuetab[pid].qnext = EMPTY;
  return pid;
}
```

## 2.9 Priority Queue Manipulation
([top](#directory))

### What is a Priority Queue?

- each item as a key value, and the queue is sorted on those values
- thus, insertion might be in the midle of the queue
- Contrast with FIFO

### `insert()`

```C
// insert - insert a process into a queue in descending key order

status insert (
  pid32     pid, // Id of process to insert
  qid16     q,   // ID of queue to use
  int32     key, // key for the inserted process
){
  int 16 curr; // runs through items in a queue
  int 16 prev; // hold previous node index

  if (isbadqid(q) || isbadpid(pid)){
    return SYSERR;
  }

  curr = firstid(q);
  while (queuetab[curr].qkey >= key){
    curr = queuetab[curr].qnext;
  }

  // insert process between curr node and previous node
  prev = queuetab[curr].qprev; // get index of previous node
  // need to inser new intem before prev and curr
  // link process into the queue
  queuetab[pid].qnext = curr;
  queuetab[pid].qprev = prev;
  // set the queue
  queuetab[pid].qkey = key;
  // update prev link for following node
  queuetab[prev].qnext = pid;
  // update next link for prior node
  queuetab[curr].qprev = pid;
  return OK;
}
```

## 2.10 Xinu List Initialization
([top](#directory))

### Desing Guideline: Consider Initialization Last

- it is done only once
- queue will be used repeatedly

> **Optimize the expected case**

- in Xinu, queues are never deleted or freed
- Same initialization for all types of queues

### Design Assumptions
- head and tail of queue are in adjacent array slots
- simplifies storage requirements
- we saw this in queue.h
```c
#define queuehead(q) (q)
#define queuetail(q) ((q) + 1)
```

### `newqueue()` Part I

```c
#include <xinu.h>

// newqueue - allocate and initialze a queue in the global queue table

qid16 newqueue(void)
{
  static qid16 nextqid = NPROC; // next list in queuetab to use
  // static means scope is local, storage is global (persists)
  // static variables are stored on the heap
  qid16        q;               // id of allocated queue

  q = nextqid;
  if (q >= NQENT) { //check for table overflow
    return SYSERR;
  }
}

```

### Xinu vs FreeBSD/Linux and So On

- All are time-sharing systems
  - but have different expected workloads
- xinu is for embedded systems
  - small number of processes 
  - most parameters determined at build time
- FreeBSD et all are for interactive sysmtes
  - desktop servers
  - dynamic workloads

## What's your point?
- notice how Xinu allocates queues
- parameters of Xinu defined statically at build time
  - size of process table
  - number of queues
- some other systems will dynamically resize tables when they fill
- xinu stresses simplicity and elegance for learning purposes

### `newqueue()` Part II

```c
nextqid +=2; //incrememtn index for next call

//initialize head and tail nodes to form an empty queue
queuetab[queuehead(q)].qnext = queuetail(q);
queuetab[queuehead(q)].qprev = EMPTY;
queuetab[queuehead(q)].qkey = MAXKEY;
queuetab[queuetail(q)].qnext = EMPTY;
queuetab[queuetail(q)].qprev = queuehead(q);
queuetab[queuetail(q)].qkey = MINKEY;
return q;
```

### `newqueue()` Full
```c
#include <xinu.h>

// newqueue - allocate and initialze a queue in the global queue table

qid16 newqueue(void)
{
  static qid16 nextqid = NPROC; // next list in queuetab to use
  // static means scope is local, storage is global (persists)
  // static variables are stored on the heap
  qid16        q;               // id of allocated queue

  q = nextqid;
  if (q >= NQENT) { //check for table overflow
    return SYSERR;
  }
  nextqid +=2; //incrememtn index for next call

  //initialize head and tail nodes to form an empty queue
  queuetab[queuehead(q)].qnext = queuetail(q);
  queuetab[queuehead(q)].qprev = EMPTY;
  queuetab[queuehead(q)].qkey = MAXKEY;
  queuetab[queuetail(q)].qnext = EMPTY;
  queuetab[queuetail(q)].qprev = queuehead(q);
  queuetab[queuetail(q)].qkey = MINKEY;
  return q;
}

```


## Week 2 Live Session
([top](#directory))

### Paper Requirements Overview
- Pick a topic and problem
- research exisiting knowledge
- propose a solution
  - (try a hybrid solution)
- implement the solution

- Paper due week 9
- We will all present in week 10
  - power point presentation

- ideas:
  - OS optimized for IPFS
  - distributed file system

### Lecture

Higher priority only see A if process has higher priority
- limited resources cause priority miscues

### Queues and Lists
- fundamental for OS
- Various forms
  - fifo
  - priority lists
  - ascending and descending

### Lists and Queues in Xinu
- need a PI
- all lists are doubly linked lists
- -1 is a nonapplicable pointer in the head

- head
  - key max
  - pointer -1 (NA)
- tail
  - key min
  - pointer -1 (NA)

**key == priority** (until lecture 8)

- Head and Tail Linked
- FIFO
  - 
- Priority Based
  - 

### queue table
- more than one queue per queuetable
  - ready process
  - running process
- There is a null process (PID=0) that is in the **real** queue table
  - The theoretical queuetable does not include the null process

### List order
- If the keys are out of order, the scheduling is FIFO
  - use `enqueue()` and `dequeue()`
  - `enqueue()` places before the tail
  - modify 4 pointers
  