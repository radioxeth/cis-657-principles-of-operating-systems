# Week 3 CPU Scheduling

## Directory
- [Home](/README.md#table-of-contents)
- [Week 2 Lists and Queues](/week2/README.md#week-2-lists-and-queues)
- **&rarr;[Week 3 CPU Scheduling](/week3/README.md#week-3-cpu-scheduling)**
- [Week 4 Process Coordination](/week4/README.md#week-4-process-coordination)

### CPU Scheduling
([top](#directory))

### The Grand Illusion

- OS presents each process with the abstracion of its own processor
- true concurrency can't happen with a singly threaded, single-core CPU
- computer speed ralitive to humans makes it seem so
- still can't predict deterministic order or rate of execution

### Use of Queues
- as noted, we can run only one process at a time
- can keep a ready queue of processes that could run if we had more CPUs
- In actuality, could be multiple queues
- ordering and maintenance of queues depends on scheduling policy

### Multiprogramming and Multitasking Support
- context switching: changing the currently running process (changing execution context)
  - hardware dependent; some ISA define instructions to save processor context; others must explicitly save
  - kernel software state saved/loaded explicitly
  - optimization is a very good idea
- scheduling: deciding which process to run next
  - related to CS, in that CS just picks the process at the head of the ready queue; scheduler maintains ready queue

### Scheduling
- processes have "bursts" of activity (both CPU bursts and I/O bursts)
- we charactize processes by the ratio of these bursts
  - Lost of I/O, little CPU: I/O bound (interactive)
  - Lots of CPU, little I/O: CPU bound (noninteractive)
- Real-time processes must have subportions completed by particular deadlines; think of factory floor automation

### Preemption
- **Preemptive** scheduling policies can take a processor away from a running process
- Under **nonpreemptive** policies, a process must give up the CPU
  - explicit yield
  - requesting I/O
  - performing another blocking operation


### Timeslice (Quantum)
- the unit of time the CPU is allocated to a process
- typically 1/100 to 1/10 of a second (10 to 100ms)
- if a process uses an entire timeslice, it may lose the CPU through preemption


### Scheduling Policies I
- FCFS (first come first served)
  - Nonpreemptive; use a FIFO queue of processes
- SJF (shortest job first)
  - Preemptive: choose process with the shortes estimated burst (aka shortest remaining time)
  - Nonpreemptive: choose process with shortes estimate runtime

### Scheduling Policies II
- Round-robin
  - Preemptive
  - when a process becomes ready, insert it at the back of the queue
- priority
  - keep the ready queue sorted by key
  - Unix: lower number = better priority
  - Xinu: higher number = better priority

### Scheduling Policies III
- Priority-based round-robin
  - combine the prior two
  - rather than inserting processes at the tail of the queue (a la round-round)...
  - insert process in priority order; if there are multiple with the same priority, insert at the back of the group

### Priority-Based Round-Robit Scheduling Invariant

> At any time, the highest-priorty eligible process is running. Among processes with equal priority, scheduling is round-robin,

## 3.3 The Process Table and process.h
([top](#directory))

### What is the Process Table?
- data structure with one entry per process
  - could be hash table, linked list, array, and so on
- hold metadata describing process
  - user id, resource usage
    - (security in multiprocess system)
- Holds data that will/may be destroyed when we change process
  - think about data saved on interrupts
  - stack pointer
  - general registers

### Xinu proctab Entry: struct procent


```c
/* Definition of the process table (multiple of 32 bits) */

struct procent {		/* entry in the process table		*/
	uint16	prstate;	/* process state: PR_CURR, etc.		*/
	pri16	prprio;		/* process priority			*/
	char	*prstkptr;	/* saved stack pointer			*/
	char	*prstkbase;	/* base of run time stack		*/
	uint32	prstklen;	/* stack length in bytes		*/
    // ^^^ above this line are important for scheduling ^^^
	char	prname[PNMLEN];	/* process name				*/
	uint32	prsem;		/* semaphore on which process waits	*/
	pid32	prparent;	/* id of the creating process		*/
	umsg32	prmsg;		/* message sent to this process		*/
	bool8	prhasmsg;	/* nonzero iff msg is valid		*/
	int16	prdesc[NDESC];	/* device descriptors for process	*/
};
```

### Process State

```c
uint16	prstate;	/* process state: PR_CURR, etc.		*/

#define	PR_FREE		0	/* process table entry is unused	*/
#define	PR_CURR		1	/* process is currently running		*/
#define	PR_READY	2	/* process is on ready queue		*/
#define	PR_RECV		3	/* process waiting for message		*/
#define	PR_SLEEP	4	/* process is sleeping			*/
#define	PR_SUSP		5	/* process is suspended			*/
#define	PR_WAIT		6	/* process is on semaphore queue	*/
#define	PR_RECTIM	7	/* process is receiving with timeout	*/
```

### Process Priority


```c
pri16	prprio;		/* process priority			*/

# define INITPRIO 20   /* Initial process priority        */
```

- key used to order ready queue
- xinu priorities are explicity assigned
- FreeBSD priorities vary over time, depending on process's cpu time
- Process in Xinu start with priorty 20

### The Process's Stack

```c
char	*prstkptr;	/* saved stack pointer			*/
char	*prstkbase;	/* base of run time stack		*/
uint32	prstklen;	/* stack length in bytes		*/
```

- each process's stack is set to an initial state
- save the stack pointer on context switch, interrupt and so on
- the saved stack pointer should always be between (`prstkabase`) and (`prstkabase`+`prstklen`)

### Some Stack Constants

```c
#define	INITSTK		65536	/* initial process stack size		*/

/* Marker for the top of a process stack (used to help detect overflow)	*/
#define	STACKMAGIC	0x0A0AAAA9

#define	INITRET		userret	/* address to which process returns	*/
```

### Other Important Declarations

```c
#ifndef NPROC
#define	NPROC		8
#endif		
```
```c
extern	struct	procent proctab[];
extern	int32	prcount;	/* currently active processes		*/
extern	pid32	currpid;	/* currently executing process		*/
```

## The Xinu Scheduler:`resched()`
([top](#directory))

### `resched()`
- function in Xinu that performs scheduling
- xinu uses a single ready list (`readylist`)
- highest-priority process at head of list
  - constant time removal
  - O(n) insertion
- priority-based round-robin (`prprio` field)
- current process (`currpid`) is not on the ready list

### Chosing Process State
- process is still eligible to run, then its state will become `PR_READY`
- if not, is will become something else (like `PR_WAIT`)
  - must not go on the ready queue
- in Xinu, scheduler (`resched()`) is not called with argument containing new state for current proc
  - argument is passed implicitly

### Implicit Argument Passing
- before calling `redched()` caller sets `prstate` field in `proctab[currpid]`
  - if current process is stil eligible to run, its state is left unchanged
- `resched()`
  - checks state of current proc, and acts accordingly
  - makes target proc ready to run
  - calls `ctxsw` (context switch routine)

### Example

- The highest-priority process, P, in the system waits on a semaphore and blocks
  - `wait()` sets P's state to `PR_WAIT`
  - `wait()` calls `resched()`
- later the semaphore is signaled
  - `signal()` makes Ps state `PR_READY`
  - `signal()` inserts P into ready list
  - `signal()` calls `resched()`

### Code for `resched()`


```c
/* resched.c - resched */

#include <xinu.h>

/*------------------------------------------------------------------------
 *  resched  -  Reschedule processor to highest priority eligible process
 *------------------------------------------------------------------------
 */
void	resched(void)		/* assumes interrupts are disabled	*/
{
	struct procent *ptold;	/* ptr to table entry for old process	*/
	struct procent *ptnew;	/* ptr to table entry for new process	*/

	/* If rescheduling is deferred, record attempt and return */

	if (Defer.ndefers > 0) {
		Defer.attempt = TRUE;
		return;
	}

	/* Point to process table entry for the current (old) process */
	ptold = &proctab[currpid];

    // if currpid is stil eligble to run
	if (ptold->prstate == PR_CURR) {  /* process remains running */
		if (ptold->prprio > firstkey(readylist)) {
			return;
		}

		/* Old process will no longer remain current */

		ptold->prstate = PR_READY;
		insert(currpid, readylist, ptold->prprio);
	}

	/* Force context switch to highest priority ready process */

	currpid = dequeue(readylist);
	ptnew = &proctab[currpid];
	ptnew->prstate = PR_CURR;
	preempt = QUANTUM;		/* reset time slice for process	*/
	ctxsw(&ptold->prstkptr, &ptnew->prstkptr);

	/* Old process returns here when resumed */

	return;
}
```

## 3.5 Deferred Rescheduling
([top](#directory))

```c
if (Defer.ndefers > 0) {
    Defer.attempt = TRUE;
    return;
}
```
- sometimes we don't want to imediately switch to another process
- eg a device such as the disk might read several sectors at once, satisfying multiple processes' requests
- we want to handle all of them before we actually `resched()`


#### in 2019 Xinu we are working wtih - `resched`===`sched`

### resched.h
```c
/* resched.h */

/* Constants and variables related to deferred rescheduling */

#define	DEFER_START	1	/* start deferred rescehduling		*/
#define	DEFER_STOP	2	/* stop  deferred rescehduling		*/

/* Structure that collects items related to deferred rescheduling	*/

struct	defer	{
	int32	ndefers;	/* number of outstanding defers 	*/
	bool8	attempt;	/* was resched called during the	*/
				/*   deferral period?			*/
};

extern	struct	defer	Defer;
```

### code for resched_cntl.c
```c
/* sched_cntl.c - sched_cntl */

#include <xinu.h>

struct	defer	Defer;

/*------------------------------------------------------------------------
 *  sched_cntl  -  control whether rescheduling is deferred or allowed
 *------------------------------------------------------------------------
 */
status	sched_cntl(		/* assumes interrupts are disabled	*/
	  int32	def		/* either DEFER_START or DEFER_STOP	*/
	)
{

	switch (def) {

	    /* Process request to defer:				*/
	    /*	1) Increment count of outstanding deferral requests	*/
	    /*	2) If this is the start of deferral, initialize Boolean	*/
	    /*		to indicate whether resched has been called	*/

	    case DEFER_START:
        //this checks if it is the first request for deferral
		if (Defer.ndefers++ == 0) {	/* increment deferrals	*/
			Defer.attempt = FALSE;	/* no attempts so far	*/
		}
		return OK;

	    /* Process request to stop deferring:			*/
	    /*	1) Decrement count of outstanding deferral requests	*/
	    /*	2) If last deferral ends, make up for any calls to	*/
	    /*		resched that were missed during the deferral	*/

	    case DEFER_STOP:
        // this is an error check
		if (Defer.ndefers <= 0) {	/* none outstanding */
			return SYSERR;
		}
        // --Defer.ndefers is PREdecrement
		if (--Defer.ndefers == 0) {	/* end deferral period	*/
			if (Defer.attempt) {	/* resched was called	*/
				resched();	/*   during deferral	*/
			}
		}
		return OK;

	    default:
		return SYSERR;
	}
}
```

## 3.6 Context Switching
([top](#directory))

### The core of the Illusion: Context Switching
- Switching from process A to process B
- Must save all of process A's context
- Must save all of process B's context
- Remember what we saved for interrupts?

### End of `resched()`

```c
ctxsw(&ptold->prstkptr, &ptnew->prstkptr); //context switch
```


### what PC to save in `ctxsw`?

- recall that we save the PC as part of process's context
- but once we save the PC for the old process, the PC changes as `ctxsw()` continues
- Where do we want to return to?
> All calls to `ctxsw()` go through `resched()`

> So we set the saved PC to just past the call to `ctxsw()` in `resched()`

### A Note on Stacks

- All Xinu processes share an address space
- all are in memory at the same tie
- xinu uses a process's stack to save register state
  - could save it in proctab
  - Data registers, PSW, and PC

<img src="/week3/images/processStack.png" width=500 />

### What `ctxsw()` does

- push registers onto stack for the old process
- save old process's stack pointer in proctab
- load new process's stack pointer from proctab
- pop register state from stack for the new process
- return to point in new process after `ctxsw()` call

### Code for `ctxsw()`

async video has different code - for an ARM processor

``` asm
/* ctxsw.s - ctxsw */

		.text
		.globl	ctxsw

/*------------------------------------------------------------------------
 * ctxsw -  call is ctxsw(&old_sp, &new_sp)
 *------------------------------------------------------------------------
 */
ctxsw:
		pushl	%ebp		/* push ebp onto stack		*/
		movl	%esp,%ebp	/* record current SP in ebp	*/
		pushfl			/* record flags			*/
		pushal			/* save general regs on stack	*/

		/* save old segment registers here, if multiple allowed */

		movl	8(%ebp),%eax	/* Get mem location in which to	*/
					/*  save the old process's SP	*/
		movl	%esp,(%eax)	/* save old process's SP	*/
		movl	12(%ebp),%eax	/* Get location from which to	*/
					/*  restore new process's SP	*/

		/* The next instruction switches from the old process's	*/
		/*  stack to the new process's stack.			*/

		movl	(%eax),%esp	/* pick up new process's SP	*/

		/* restore new seg. registers here, if multiple allowed */

		popal			/* restore general registers	*/
		movl	4(%esp),%ebp	/* pick up ebp before restoring	*/
					/*   interrupts			*/
		popfl			/* restore interrupt mask	*/
		add	$4,%esp		/* skip saved value of ebp	*/
		ret			/* return to new process	*/
```

## 3.7 Additional Scheduling Notes 
([top](#directory))

### Process State Transition Diagram

- `resched()` moves process only between ready and current states

### Ramification
- if `resched()` can only switch the process from one process to another..
- There must always be aprocess ready to run - the NULL process!
- PID 0
- Priority 0
- Only runs when all other processes are blocked (eg semaphores, I/O)


### Maintaining the Scheduling Invariant
- remember the scheduling invariant
- every functino that changes a process state must ensure the invariant holds
  - highest-priority process was running before function
  - (possibly different) highest-priority process must be running after


### Examples
- `signal()` can unblock a process waiting on a semaphore
- how does `signal()` make a process ready to run?
  - it calls `ready(pid)`
- `wait()` might block a process
  - need to call `resched()`

### `ready()`

- mark a process as READY
- insert it in the ready list
  - ordered by priority
- call `resched()` to maintain invarriant

> if the current process has higher priority than the readied process, `resched()` is a no-op

### Code for `ready()`

```c
/* ready.c - ready */

#include <xinu.h>

qid16	readylist;			/* index of ready list		*/

/*------------------------------------------------------------------------
 *  ready  -  Make a process eligible for CPU service
 *------------------------------------------------------------------------
 */
status	ready(
	  pid32		pid,		/* ID of process to make ready	*/
	  bool8		resch		/* reschedule afterward?	*/
	)
{
	register struct procent *prptr;

	if (isbadpid(pid)) {
		return(SYSERR);
	}

	/* Set process state to indicate ready and add to ready list */

	prptr = &proctab[pid];
	prptr->prstate = PR_READY;
	insert(pid, readylist, prptr->prprio);

	if (resch == RESCHED_YES) {
		resched();
	}
	return OK;
}
```

### Another Treatment of Priority

- xinu hsa (mostly) static priorities
  - befits an OS for embedded systsm
- FreeBSD is used in Mac OS X
  - interactive workloads
  - process priorities change over time
  - goal: keep interactive processes responsive

### BSD Scheduling
- BSD uses a priority-based round-robin scheme favoring interactive processes
- Processes have dynamic priorities
- jobs get a time slice
  - complete your time slice: priority worsens
  - give up the CPU before the end: priority stays the same
  - interavtive processes have priority improved

### Ramifications of Scheduling
- long-lived CPU-bound
  - jobs drop quickly in priority
  - they'll run only when there aren't any high-priority interactive jobs available
  - but they won't starve (sometimes get more priority)
- interactive jobs
  - maintain high priority
  - don't monopolize CPU
- mixed-mode jobs
  - drop in priority when CPU-bound
  - Rise back up later

### FreeBSD Process Priority Calculation
- from proc.h
  - `kg_estcpu` provides estimate of recent CPU utilization
  - `kg_nice` user-settable weighting factor between -20 and 20 (default 0)

- Recalculate every four ticks

```c
kg_user_pri = PRI_MIN_TIMESHARE + (kg_estcpu/4) + 2*kg_nice;
```

- `kg_user_pri=max(kg_user_pri, PRI_MIN_TIMESHARE)`
- `kg_user_pri=min(kg_user_pri, PRI_MAX_TIMESHARE)`
- `PRI_MIN_TIMESHARE`=160 
- `PRI_MAX_TIMESHARE`=223

- `kg_estcpu` incremented every clock tick for the current process
- `kg_estcpu` aged (decayed) every second; forgets about 90% of history in a variable time period

```c
load_weight = (2*load)/(2*load+1);
kg_estcpu = load_weight * kg_estcpu + kg_nice;
```

## Week 3 Live Session

### Concurrent processing

### Queue Table vs Process Table

- queue table
  - place to store **ready processes**

- process table
  - place to store **all processes**

```
$ ps
// prints the process table
```

### Process State

- current (currently executing)
- ready (process is ready to execute)
- waiting
- receiving
- sleeping
- suspended

```c
/* Process state constants */

#define	PR_FREE		0	/* process table entry is unused	*/
#define	PR_CURR		1	/* process is currently running		*/
#define	PR_READY	2	/* process is on ready queue		*/
#define	PR_RECV		3	/* process waiting for message		*/
#define	PR_SLEEP	4	/* process is sleeping			*/
#define	PR_SUSP		5	/* process is suspended			*/
#define	PR_WAIT		6	/* process is on semaphore queue	*/
#define	PR_RECTIM	7	/* process is receiving with timeout	*/
```

- Scheduling only looks at eligable processes
  - current
  - ready
- context switch
  - moves current process into ready queue
  - stack is in the memory, takes time
    - limited context switch
- FIFO is fair schedule
- processes with equal priority undergo round robin

### Deferred Rescheduling

- atomic code
  - turns off the scheduling
  - some real-time operating systems need to perform operations in a set amount of time. But if they miss the time window, there is no reason to execute to process because it's too late.

### For lab 4
- to change scheduling technique to FIFO
  - replace `insert()` in ready.c with `enqueue()`


### Semaphore

- concept used for resource allocation

```
sid32 variable; declare the semaphore variable
variable = semcreate(number);
wait(variable); decrement number by 1
signal(variable); increment the number by 1
```