# Week 7 Interrupts and Devices


## Directory

- [Week 6 Message Passing](/week6/README.md#week-6-message-passing)
- **&rarr;[Week 7 Interrupts and Devices](/week7/README.md#week-7-interrupts-and-devices)**
- [Week 8 Devices](/week8/README.md#week-8-devices)

## 7.2 Interrupts in Depth
([top](#directory))

### recall
- interrupts allow IO and computation to occur in parallel
- vectored interrupts use a lookup table
- handler found in `interrupt_vector[i]`
- hardware saves some state
- handler must save the rest
- exceptions are handled with much the same mechanism

### Interrupt Vector

|||
|-|-|
|||

### Examples of Exceptions

- page faults
- diviide by 0
- illeagal instructions
- protection violations
- illegal memory references
- unused
- two device interrupts for IO
    - IRQ
    - FIQ

### Requirements for ARM IRQ exception-handling code
- the ARM architecture requires a two-level interrupt-handling scheme
- all devices raise the same exception (IRQ)
  - IRQ is more general and FIQ; Xinu doesn't use FIQ
- Therefore IRQ exception handling code must:
  - query hardware to find IRQ of interrupting device
  - find the code that handles that particular device
    - usually using a second lookup table

### ARM Exception Handling
- exception_vector[i] contains the code to be executed for exception i
- not a pointer to a function; the actuale code!
- and each entry is only 4 bytes long (one instruction)
- therefore each entry must contain a jump
  - use indirect branch instructions
  - branch to address sorted in another memory location

<img src="/week7/images/interrup-table.png" width=500>

### Exception Vectors and Array of Pointers in Contiguous Memory

|x|x|x|&uarr;|&uarr;|&uarr;|
|-|-|-|-|-|-|
|PC|||PC+24|||

`X = idr pc, [pc #24]`

3 exception vectors
3 pointers

## 7.3 Interrupt Dispatching
([top](#directory))

- dispatching comprises four steps:
  1. Obtain device number
  2. index into vector
  3. Extract handler address
  4. Pass control to handler
- dispatcher written in assmebler
- handler written in C
- harware support: interrupt controller

### ARM Interrupt Controller
- handles communications over the bus
- does not store interrupt vectors
- on interrupt, the controller:
  1. obtains integer IRQ from device
  2. raises IRQ exception
  3. passes IRQ to processor
- os must use IRQ as index in interrupt vector and call the handler

### Conceptual Structure of Interrupt Software

### Dispatching
- review interrupts video from week 2
- on the way in 
  - push registers on to stack
  - call handler
- on the way out
  - pop registers from stack
  - return from interrupt


### Two level interrupt handling on the ARM
- interrupt controller jumps to exception handler (which is an indirect jump to irq_dispatch)
    - branch into irq_dispatch
- irq_dispatch retrieves IRQ and looks up IRQ in table, branching to correct handler for that type of device
    - calls the handler
- device handler processes the interrupt
    - for example, the ethernet handlre reads a packet

### questions

- `IRQ expection handler` is an indirect jump to irq_dispatch
- and calls the `interrupt handler` for the interrupting device
- from the interrupt controller, users it as an index into the `interrupt vector`
- during the second stage, irq_dispatch fetches the `IRQ`

## 7.4 Disabling Interrupts
([top](#directory))

- all architectures have a setting where interrupts are (temprarily) ignored
  - ignore all interrupts
  - prioritize interrupts and ignore all those below a certain level
- we've seen how to use this to provide mutex on a uniprocessor
- hardware automatically disables further interrupts when an interrupt occurs

### Durraction of automatic disabling
- starts when interrupt occurs
- continues while
  - dispatcher runs
  - handler is called
  - handler returns
- ends when dispatcher executes special return from interrupt instruction

> Result: interrupt handling code cannot be interrupted

### Consequences
- if interrupts are off too long, devices will have errors
  - network might drop packets
  - clock could lose time
- time (spee) of handling is of the essence
- disabling interrupts to handle on device may affect all devices
> the upper limit on time for an interrupt handler is determined not by that device but rather by the most delay-sensitive device in the system

### Function Invocation
- interrupt code must be executable by an arbitrary process
- an interrupt handler can invoke operating system functions (including those that change process state)
- the scheduler assumes there will always be a ready process
  - this is the purpose of the null process

### Clarifying Example
- suppose there are only two processes in the system
  - process A
  - null process
- we know that the null process will run only when A is blocked
- A reads from the network
  - will block until a packet arrives

- when process A blocks, the null process runs
- time passes and the null process loops
- bang! packet arrives
- the only process that can possibly be running then is the null rpocess
- but the null process can never block

> result: interrupt routines can only call OS functions that leave the current process in the CURRENT or READY states

### Should we allow rescheduling during interrupt handling
- to maintain the scheduling invariant, we mush
- consider our clarifying example
- if we return to the nullprocess without rescheduling, A will never run again!

> To ensure that processes are notified promptly when an I/O operation completes and to maintain the scheduling invariant, an interrupt handler must reschedule whenever it makes a waiting process ready *- text, page 226*

### Why and When is this OK?
- resched() will no block the current process
- before calling resched(), the handler must make sure global state is consistent
  - doesn't  know when it will get a chance to run again
- all OS functions* restore, not enable, interrupts before returning.
> *the only ode that explicitly enables interrupts is the system startup code


## 7.5 Time and Timers
([top](#directory))

### Why do we need timers?
- applications might want to dipslay a message for a few seconds
- we use time-outs to detect errors
  - no respone from webserver
- We can track the time of day

### Four types of time-related hardware

- processor clock
  - the clock determines execution speed
  - when we say processor executes at 2.4 GHz
  - roughly one tick per instruction
  - Does not generate interrupts
- real-time clock
  - pulses in fractions of a second (1 kHz)
  - generates an interrupt on each pulse
  - does not count pulses
- Time of day clock
  - typically built over real-time clock, tallying pulses
  - does not interrupt
- Interval timer
  - aka counter to value
  - value decremented at each real-time clock pulse
  - interrupt when value drops to zero

### RTC vs Interval Timer

- RTC interrupts on every pulse
- timer interrupts only when specified period has elapsed
- timer can emulate an RTC
  - for pulse rate R pulses/second, set interval to 1/R seconds
  - reset timer to identical period every interrupt
  - abstraction in action

### Not all systems have all clocks
- it's common for a system to only have a subset of four clocks
- galileo has an RTC
  - xinu configures it for a 1-milliseconds pulse
- beaglebone black has an interval timer
  - xinu sets timer to interrupt after 1 ms
  - when handling timer interrupt, reset timer to 1 ms

> xinu is built on the abstraction of a 1-millisecond RTC

### Speed Wins
- must handle a clock interrupt before the next pulse arrives
  - Otherwise we will miss it (interrupts disabled)
  - source of hidden timing errors
- luckily the processor clock is many times faster than the RTC
  - 1kHz RTC
  - 2.4 GHz processor clock
  - 2,400,000 clock ticks per RTC pulse

## 7.6 Preemption

([top](#directory))

### Uses of Timed Events
- Timed delay
  - any process can set a timer
  - process enters the sleeping state
  - after delay, wakeup occurs
- preemption
  - give the CPU to a proces sin increments called quanta or time slices with length T
  - when switching to a process, scheduler sets a timer event for T time units in the future
  - event handler calls `resched()`

### Time Slicing and Preemption
- consider a set of equal-highest-priority CPU-bound processes
- scheduling invariant says we should do round-robin between them
- each of them will execute to the end of its quantum
- interval timer will interrup, reset the timer, and call `resched()`

### Length of Time Slice
- how long should the quantum be?
- long enough to keep relative overhead low
  - interrupt handling and context switching are overhead
- short enough to keep processes are well-served
- in real systems, this isn't much of an issue
  - processor rarely execute full time slice
  - I/O or calls blocking function
-
### why have preemption?

> Without a preemptive capability, an operating system cannot regain control from a process that executes an infinite [CPU-bound] loop - *text page 237*

infinite loops are usually programming errors.

## 7.7 Implementing Preemption and Delay
([top](#directory))

- QUANTUM: nimber of ticks in a time slice
- When switching processes, `resched()` sets global variable preempt to QUANTUM
- clock handler decrements preempt on every clock tick
- when preempt = 0, set preempt = QUANTUM and call `resched()`
- then return from interrupt

### Wait a tick...
- why did the interrupt handler for the clock set preempt = QUANTUM
- Wouldn't `resched()` always do that?
- No - `resched()` resets preemption only when switching processes
- if we have just one highest priority process, it keeps running
- this prevents underflow of preempt

### Implementing Delay
- Multiple processes can request delay timers
- requests are relative to the time when request was made
- need efficient data structure to store requests
- can't afford long searches on each tick
- solution: delta list

### Delta List
- list of processes
- sorted in order of time to awaken
- use relative, not absolute, times
- use relative, not absolute, times
- key of first processes is remaining ticks from now to delay
- key of each other process specifies additional ticks beyond preceding process

### Example
- four processes make delay requests within the same clock tick
- A requests delay of 6
- B requests delay of 12
- C requests delay of 27
- D requests delay of 50

|6|A|&rarr;|6|B|&rarr;|15|C|&rarr;|23|D|&rarr;|^|
|-|-|-|-|-|-|-|-|-|-|-|-|-|

### Delta List Insertion: `insertd()`

- uses `queuetab` (compare ready list)
- recall that each entry has a key and a next field (proc id is the index)
- global sleep queue: sleepq (in clkinit.c)
- walk list, subtracting keys from desired delay
- stop when desired delay is < next key
- insert process
- Adjust key of next process


### `insertd()`
```c
/* insertd.c - insertd */

#include <xinu.h>

/*------------------------------------------------------------------------
 *  insertd  -  Insert a process in delta list using delay as the key
 *------------------------------------------------------------------------
 */
status	insertd(			/* assumes interrupts disabled	*/
	  pid32		pid,		/* ID of process to insert	*/
	  qid16		q,		/* ID of queue to use		*/
	  int32		key		/* delay from "now" (in ms.)	*/
	)
{
	int	next;			/* runs through the delta list	*/
	int	prev;			/* follows next through the list*/

	if (isbadqid(q) || isbadpid(pid)) {
		return SYSERR;
	}

	prev = queuehead(q);
	next = queuetab[queuehead(q)].qnext;
	while ((next != queuetail(q)) && (queuetab[next].qkey <= key)) {
		key -= queuetab[next].qkey;
		prev = next;
		next = queuetab[next].qnext;
	}

	/* Insert new node between prev and next nodes */

	queuetab[pid].qnext = next;
	queuetab[pid].qprev = prev;
	queuetab[pid].qkey = key;
	queuetab[prev].qnext = pid;
	queuetab[next].qprev = pid;
	if (next != queuetail(q)) {
		queuetab[next].qkey -= key;
	}

	return OK;
}
```


## 7.8 Sleeping
([top](#directory))

### Updated Process State Diagram
- see main page

### How Processes Delay
- process don't call `insertd()` directly
- they call `sleep()` or `sleepms()`
  - `sleep(sec)`: sleep for `sec` seconds
  - `sleepms(msec)`: sleep for `msec` milliseconds
  - `sleep()` actually multiplies `sec x 1000` and calls `sleepms()`
- `sleepms()` calls `insertd()`
- 32-bit unsigned integer: about 49 days

### Code for `sleepms()`

```c
/*------------------------------------------------------------------------
 *  sleepms  -  Delay the calling process n milliseconds
 *------------------------------------------------------------------------
 */
syscall	sleepms(
	  uint32	delay		/* time to delay in msec.	*/
	)
{
	intmask	mask;			/* saved interrupt mask		*/

	mask = disable();
	if (delay == 0) {
		yield();
		restore(mask);
		return OK;
	}

	/* Delay calling process */

	if (insertd(currpid, sleepq, delay) == SYSERR) {
		restore(mask);
		return SYSERR;
	}
	sltop = &queuetab[firstid(sleepq)].qkey;
	slnonempty = TRUE;
	proctab[currpid].prstate = PR_SLEEP;
	resched();
	restore(mask);
	return OK;
}
```
### `yield()`

yield disables interrupts and then calls `resched()` because `resched()` assumes disabled interrupts

```c
/* yield.c - yield */

#include <xinu.h>

/*------------------------------------------------------------------------
 *  yield  -  Voluntarily relinquish the CPU (end a timeslice)
 *------------------------------------------------------------------------
 */
syscall	yield(void)
{
	intmask	mask;			/* saved interrupt mask		*/

	mask = disable();
	resched();
	restore(mask);
	return OK;
}
```

## 7.9 Message Time-Outs
([top](#directory))

### time-outs
- often we want to block until either
  - an event occurs
  - a time period has passed
- example: connecting to a web server
- can achieve this through timed message reception
  - if event occurs, handler sends process a message
  - if even doesn't happen, then time-out occurs

### new call: `recvtime()`

- timeout is in ticks (msec)
- first part lloks like `sleepms()`, except state is `PR_RECTIM` (vs `PR_SEEP`)
- after being awakened, check whether a message has arrived
- if not, then there must have been a timeout

```c
/* recvtime.c - recvtime */

#include <xinu.h>

/*------------------------------------------------------------------------
 *  recvtime  -  wait specified time to receive a message and return
 *------------------------------------------------------------------------
 */
umsg32	recvtime(
	  int32		maxwait		/* ticks to wait before timeout */
        )
{
	intmask	mask;			/* saved interrupt mask		*/
	struct	procent	*prptr;		/* tbl entry of current process	*/
	umsg32	msg;			/* message to return		*/

	if (maxwait < 0) {
		return SYSERR;
	}
	mask = disable();

	/* Schedule wakeup and place process in timed-receive state */

	prptr = &proctab[currpid];
	if (prptr->prhasmsg == FALSE) {	/* if message waiting, no delay	*/
		if (insertd(currpid,sleepq,maxwait) == SYSERR) {
			restore(mask);
			return SYSERR;
		}
		sltop = &queuetab[firstid(sleepq)].qkey;
		slnonempty = TRUE;
		prptr->prstate = PR_RECTIM;
		resched();
	}

	/* Either message arrived or timer expired */

	if (prptr->prhasmsg) {
		msg = prptr->prmsg;	/* retrieve message		*/
		prptr->prhasmsg = FALSE;/* reset message indicator	*/
	} else {
		msg = TIMEOUT;
	}
	restore(mask);
	return msg;
}
```

## 7.10 Awakening Processes
- on time-out a process will be at the head of the queue, with key 0
- Others behind it might need to be awakend as well
  - delta key value also of 0
- on message receipt, process will have a non-zero key value
- a process being killed migh be on sleep queue

### `unsleep(pid)`
- removes process form the sleep queue before a time-out 
  - called by `send()`
  - called by `kill()`
- adjust key of following process
- remove process from queue

```c
/* unsleep.c - unsleep */

#include <xinu.h>

/*------------------------------------------------------------------------
 *  unsleep  -  Remove a process from the sleep queue prematurely by
 *			adjusting the delay of successive processes
 *------------------------------------------------------------------------
 */
syscall	unsleep(
	  pid32		pid		/* ID of process to remove	*/
        )
{
	intmask	mask;			/* saved interrupt mask		*/
        struct	procent	*prptr;		/* ptr to process' table entry	*/

        pid32	pidnext;		/* ID of process on sleep queue	*/
					/* that follows the process that*/
					/* is being removed		*/

	mask = disable();

	if (isbadpid(pid)) {
		restore(mask);
		return SYSERR;
	}

	/* Verify that candidate process is on the sleep queue */

	prptr = &proctab[pid];
	if ((prptr->prstate!=PR_SLEEP) && (prptr->prstate!=PR_RECTIM)) {
		restore(mask);
		return SYSERR;
	}

	/* Increment delay of next process if such a process exists */

	pidnext = queuetab[pid].qnext;
	if (pidnext < NPROC) {
		queuetab[pidnext].qkey += queuetab[pid].qkey;
	}

	if ( nonempty(sleepq) ) {
		sltop = &queuetab[firstid(sleepq)].qkey;
		slnonempty = TRUE;
	} else {
		slnonempty = FALSE;
	}
	getitem(pid);			/* unlink process from queue */
	restore(mask);
	return OK;
}
```

### `wakeup()`

- this is the simple case
- awaken all processes at the front of the queue with key=0

```c
/* wakeup.c - wakeup */

#include <xinu.h>

/*------------------------------------------------------------------------
 *  wakeup  -  Called by clock interrupt handler to awaken processes
 *------------------------------------------------------------------------
 */
void	wakeup(void)
{
	/* Awaken all processes that have no more time to sleep */

	while (nonempty(sleepq) && (firstkey(sleepq) <= 0)) {
		ready(dequeue(sleepq), RESCHED_NO);
	}
	
	if ( (slnonempty = nonempty(sleepq)) == TRUE ) {
		sltop = &queuetab[firstid(sleepq)].qkey;
	}
	resched();
	return;
}
```

## Live Session

TIMEOUT is different than OK.
OK means message was not received
TIMEOUT means 

### open,read,write,close
 standard operations

 Early binding, you have to tell xinu that you will connect the IO device

### BUffer
Devices always produce (even if its empty space in the buffer)
OS always consumes