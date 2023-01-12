# Week 1 Introduction and Review, Concurrency, and Shared Memory

## Directory
- [Home](/README.md#table-of-contents)
- **&rarr;[Week 1 Introduction and Review, Concurrency, and Shared Memory](/week1/README.md#week-1-introduction-and-review-concurrency-and-shared-memory)**
- [Week 2 Lists and Queues](/week2/README.md#week-2-lists-and-queues)

## 1.2 Two Core Operating System Purposes
([top](#directory))

### Purpose 1: Abstraction

- provide new funtionality
  - build capabilities not directly supported by underlying mechanisms
  - extend existing capabilities
- masking details
  - hide complexity
  - more pleasant environment

### Purpose 2: Arbitration

- manage multiple demands for a scarce resource
  - who gets to use it now?
  - what does "fairness" mean?
- ensure proprietary of access.
  - privacy/confidentiality
  - integrity

### Conclusion

- abstraction: idealized view of an artifact
- arbitration: controls of access to a resouce
- as we go thorugh the course, look fo examples of abstractoin and arbitration in the system

## 1.3 Operating System Organization
([top](#directory))

### Overview
- what is and isn't an operating system
- organization through layers
- operating system services
- xinu code

### what is an operating system?
- abstraction and arbitration
- it's a set of programs that provides services and manages hardware

### What is it NOT?

- a language or compier
  - os are written using them
- user interface (windowing system, broswer, cli)
  - ui uses facilities in the underlying OS to perfrom tasks as directed by user
- library of functions
  - although libraries can wrap OS services

### This is confusing
- rudimentary computing systems didn't have a separate OS 
  - on program at a time, program included code to handle hardware
- microsoft claimed IE was part of Windows (and embed the GUI in the OS)
- Java has an integrated runtime system for its Virtual Machine, but the JVM runs on OS

### Where to Draw the Line
- want to distinquish kernel and user code
- kernel code:
  - shared (one instance per system/subsystem)
  - trusted
  - runs in high-privilege mode
- user code:
  - also called application code
  - multiple instances

### Structuring the OS: Layers
- hardware is the cores
- each layer further abstracts hardware, adds services
- layers can use facilities in any lower layer
- thus applications can access all system services

### OS Services
- Built upon, but not identical to, the layers
- concurrent execution
- prcoess/thread synchronization
- inerprocess communication
- dynamic memory allocation

- address space management and virtual memory
- abstracted device interfaces
- networked communication
- file systems (long-term storage)

### How to access services
- From inside the kernel: function call
- from outside the kernel: system call
  - put system call number in register or on stack
  - software trap (details to come)
- Crossin gthe user/kernel boundary
  - change in processor mode
  - change in address space
  - change in namespace

### Example service: `putc()`

```c
/* ex1.c - main */
#include <xinu.h>

/*
* main - write "hi" to the console
*/

void main (void){
    putc(CONSOLE, 'h');
    putc(CONSOLE, 'i');
    putc(CONSOLE, '\n');
}
```

## 1.4 Concurrency Models
([top](#directory))

### Dealing with Multiple Activities
- We expect computers to more than one thing at once
  - or at least appear to
  - if they only had one thing to do, simple: Poll
- Consider a smartphone
  - handle ongoing phone call
  - receive text message
  - user adjusts volume

### Model 1: Synchronous Event Loop

```

while (1){
update time of date clock
if( screen timeout has expired)
  {turn off screen;}
if(volume button is being pushed)
  {adjust volume;}
if(text message has arrived)
  {display notification to user;}
}
```

- entire event loop must bee process quickly or an event might be lost


### Model 2: Asynchronous Event Handlers
- hardware directly invokes routines (handlers) to handle events
- handlers at known locations (e.g. 0x1000)
- hardware configured to jump to appropriate location
- handlers coordinate through global variables
- brittle: programmer must coordinate all handlers and configure hardware correctly

### Model 3: Concurrent Execution
- System organized as a set of programs
  - possibly cooperating, but not tightly linked
- programs have the illusion of running concurrently
  - true concurrency requires multiple processors
- each program runs until it finishes
- more powerful and easy to user than sync or async events

### Basics of Concurrent Processing
- sequential: a single program executing one statement after another
- concurrent: multiple programs running simultaneously
- obviously, N processors can run N programs
- But how to run N programs on a single processor concurrently?
  - we can't but we can fake it

### multitasking
- provides the illusion of concurrency
- switch from one process to the next every few milliseconds
- toofast for humans to detect
- plenty of time for computation
- can be mixed with true concurrency (run N programs on M processors, N>M)
- two types of multitasking: timesharing and real-time processing

### Timesharing
- mixed loads: interactive and compute-bound programs
- processes created dynamically
- what we use every day
  - running pwerpoint, email, web browser, music player etc

### Real-Time
- (typlically) a small set of fixed prcoesses
- computations must complete within a fixed time bound
- Examples:
  - aricraft control
  - factory floor automation
  - embedded systems
  
## 1.5 Concurrency Part 2: Concurrent Processes
([top](#directory))

### Sequential vs Concurrent

|Sequential|Concurrent|
|-|-|
|Program can be thought of in isolation|OS code is concurrent|
|No concerns about interference|Multiple processes can run the same code (invoke OS function)|
|No shared data|Multitasking switches between processes|
|No shared code|no guarentees about order or rate of execution|
|Most user-level apps are sequential||

### Concurrent Processes in Xinu

```c
#include <xinu.h>
//main example of creating processes in Xinu

void sndA(void), sndB(void)

void main(void){
  resume(create(sndA, 1024, 20, "p 1", 0));
  resume(create(sndB, 1024, 20, "p 2", 0));
}

//sndA repeatedly emit 'A' on console
void sndA(void){
  while(1)
    putc(CONSOLE, 'A');
}

//sndB repeatedly emit 'B' on console
void sndB(void){
  while(1)
    putc(CONSOLE, 'B');
}
```

### Invoking Code in Xinu
- functions can either be called or created a processes
- functions created as a processes must return void
- create forms a new process with the specified code body and arguments, in suspended state

```
pid = create(sndA, 1024, 20, "p 1", 0);
```

|pid=|create(|sndA|,1024|,20|,"P 1"|0);|
|-|-|-|-|-|-|-|
|returns process Id||function address|stack size|priority|name|# function args|

- resume makes a suspended process ready to run

### What main() does, step 1
- main creates a process embodying sndA and starts it running
### What main() does, step 2
- main creates a process embodying sndB and starts it running

### Concurrent Processes
- once we `resume()` a process from `create()`, it is running at the same tie we (the caller of resume) are
- process share the CPU (details in scheduling)
- no guarantees on order of execution, lenght of execution, before another process runs

### What main() does, step 3
- main has reached the end of its function, so the process exits
- the processes for sndA and sndB continue to run

### Key Points
- concurrent processes can interfere with one another
- no guarantees of order or rate of execution
- Xinu creates suspended process from functions
- Other OS (such as Unix) clone a running process then load a program from a file

## 1.6 Shared Memory and Race Conditions
([top](#directory))


### Alternative to Last Example
```c
#include <xinu.h>
//main example of creating processes in Xinu

void sndch(char ch);

void main(void){
  resume(create(sndch, 1024, 20, "Send A", 1, 'A'));
  resume(create(sndch, 1024, 20, "Send B", 1, 'B'));
}

//output a character to console repeatedy
void sndch(char ch){
  while(1)
    putc(CONSOLE, ch);
}
```

### How is this Different?

- Separate functions don't share local variables or arguments
- how do we make it so that two process instance of the same function don't corrupt eachother?

> Each process has a private copy of local variables, arguments, and a function call stack

### Where are those stored?
- local variables and arguments are stored on the process's stack
- stack grows and shrinks as function calls, and returns happen during execution
- stack is destroyed when process exits.

> This is a key difference between functino call and process creation (possibly based on the same function)

### What about Global Variables?
- global variables are shared between all Xinu processes
  - all Xinu processes are kernel processes
  - share the same address space
  - similar to kernel threads, in eg FreeBSD
- What might happen if two concurrent threads access share global variables

### Uncoordinated Concurrent Processes

```c
#include <xinu.h>

void produce(void), consume(void);

int32 n = 0; //shared global

//main unsynch producer consumer

void main(void){
  resume(create(consume, 1024, 20, "c", 0));
  resume(create(produce, 1024, 20, "p", 0));
}

// produce increment n 2000x
void produce(void){
  int32 i;

  for (i=0;i<2000;i++)
    n++;
}

// consume print n
void produce(void){
  int32 i;

  for (i=0;i<2000;i++)
    printf("n=%d\n",n);
}
```

### possile output values, choice 1

0, 1, 2, ... , 1998, 1999, 2000

### choice 2

0, 0 ,0 , 800, 1001, 1852, 2000, ..., 2000

### choice 3

0, 0, 0,..., 0, 2000,..., 2000

produce is much faster than consumer!
computation is much faster than I/O

### what happens?
1. main creates consumer
2. main creates producer
3. consumer runs some (0,0,0,0...0)
4. Producer runs completely (n=2000)
5. Consuer finishes (2000,2000,...,2000)

This is a simple form of race condition: unctrolled access to a shared variable

### How can we solve this?

- produce could produce an item
- repeatedly check if consumer has consume
- consumer can consume an item
- repeatedly check if producer has procuded

- BUSY WAITING
  - BAD!!
  - most of the time

### Semaphores
- synchronization mechanism introduced by Djikstra in 1962
- and integer counter with operations
  - wait: decrement counter; if <0 block
  - single: increment counter; if >=0 resume a process blocked on the semaphore
- value of semaphore
  - if >=0: how many resources available
  - if <0: how many processes blocked

### Semaphores II
- avoid busy waiting
- most general: counting semaphore
- Xinu operation: `sid = semcreate(initval);`
- Mutex (mutual exclusion) semaphore: only one process allowed to access resource at a time

### Key Points
- each process has private local variables, arguments, and stack
- all processes share global variables
- willy nilly access to global variables can cause race conditions
- Arbitrate access to global variables through locks (semaphores in Xinu)