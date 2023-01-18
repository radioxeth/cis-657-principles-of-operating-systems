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
