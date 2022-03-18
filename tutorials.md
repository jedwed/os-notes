<h1>Tutorials</h1>

---

- [Week 2](#week-2)
  - [Intro](#intro)
    - [Processor Modes](#processor-modes)
    - [Role of the Operating System](#role-of-the-operating-system)
    - [Protected operations](#protected-operations)
  - [Operations and syscalls](#operations-and-syscalls)
  - [Processes and Threads](#processes-and-threads)
    - [Three-state process model](#three-state-process-model)
      - [Transitions](#transitions)
    - [Concurrency](#concurrency)
- [Week 3](#week-3)
  - [Synchronisation](#synchronisation)
    - [Q1: Semaphores](#q1-semaphores)
    - [Q2](#q2)
    - [Q3: Conditional Variables](#q3-conditional-variables)
  - [Deadlock](#deadlock)
    - [Q4](#q4)
    - [Q5 (Got this wrong)](#q5-got-this-wrong)
  - [Concurrency and Deadlock](#concurrency-and-deadlock)
    - [Q9](#q9)
- [Week 4](#week-4)
  - [R3000 and Assembly](#r3000-and-assembly)
    - [Q1](#q1)
    - [Q2](#q2-1)
    - [Q3](#q3)
    - [Q4](#q4-1)
  - [Threads](#threads)
    - [Q7](#q7)
    - [Q8](#q8)
  - [Kernel Entry and Exit](#kernel-entry-and-exit)
    - [Q9](#q9-1)
    - [Q10](#q10)
    - [Q11](#q11)
    - [Q12](#q12)
    - [Q14](#q14)
    - [Q15](#q15)
    - [Q16](#q16)

---

# Week 2

## Intro

### Processor Modes
*What are differences between user mode and priveliged mode (kernel modes)*

In user mode, the following resources should be unavailable
- CPU control registers (protected mode, interrupt control)
- CPU management instructions (eg. context switch, hogging CPU)
- Protected parts of the address space (containing kernel code or data)
- Some device memory and registers (or ports) are inaccessible

The two modes exist to ensure user-level applications can't bypass, circumvent or take control of the operating system


### Role of the Operating System
- An Operating System acts as an abstract machine that allows user-level applications to run whilst hiding the details of the hardware
- It also acts as a resource manager which shares the resources (memory, CPU) fairly between all applications (and users)


### Protected operations
*Which of the following instructions should only be allowed in kernel mode?*

- [x] Disable all interrupts
- [ ] Read time of day clock
- [x] Set the time of day clock
- [x] Change the memory map
- [x] Write to the hard disk controller register
- [ ] Trigger the write of all buffered blocks associated with a file back to disk (fsync)

---

## Operations and syscalls

```c++
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char teststr[] = "The quick brown fox jumps over the lazy dog.\n";

main()
{
  int fd;
  int len;
  ssize_t r;


  fd = open("testfile", O_WRONLY | O_CREAT, 0600);
  if (fd < 0) {
    /* just ungracefully bail out */
    perror("File open failed");
    exit(1);
  }
  
  len = strlen(teststr);
  printf("Attempting to write %d bytes\n",len);
  
  r = write(fd, teststr, len);

  if (r < 0) {
    perror("File write failed");
    exit(1);
  }
  printf("Wrote %d bytes\n", (int) r);
  
  close(fd);

}
```

1. *What does this code do?*
The code opens a file and writes to it, and creates it if it doesn't exist (`O_CREAT`)
2. *In addition to `O_WRONLY`, what are two other ways one can open a file? (Write only)*
`O_RDONLY`, `O_RDWR` (Read only, Read Write)
3. *What does open return in `fd`, what is it used for? (Consider success and failure)*
The return value gives information on whether opening the file was successful or not. It is used for error checking, to make sure the process doesn't progress without a proper open file

<br>

```bash
strace ./a.out
```
1. *What is strace?*
A system call trace
2. *Why does `printf()` not appear in strace?*
`printf()` is just a library function that creates a buffer and passes it to `write()`

---

## Processes and Threads

### Three-state process model
1. **Running**: Process is currently executing
2. **Ready**: Process is ready to execute but not yet selected by the dispatcher
3. **Blocked**: Process awaits some event/resource prior to execution

#### Transitions
- **Running->Ready**: Timeslice expired, yield, higher priority process is ready
- **Running->Blocked**: Requested resource (file, disk block, printer, mutex) is unavailable, so process is blocked until it is available
- **Blocked->Ready**: A blocked process has acquired a resource that it requires to execute, so it is queued for execution
- **Ready->Running**: Dispatcher chooses the next thread to run


### Concurrency
Can this cause race conditions?
```c++
x = x + 1;
```
Yes. It's assembly code is roughly as follows:
```assembly
LOAD temp_register, &x
ADD temp_register, temp_register, 1
SAVE temp_register, &x
```

If this code runs twice on the same x with two different threads: both threads may not be able to acquire the updated `x` value, hence the two threads may overwrite each other, causing `x` to not be correct

---

# Week 3

## Synchronisation

### Q1: Semaphores
*What synchronisation mechanism or approach might one take to have one thread to wait for another thread to update state?*
**Semaphores!**

```c++
my_sem = create_semaphore(0);

void waiting_thread(void) {
  // P(): wait
  P(my_sem);
  progress_after_state_is_updated();
}

void updating_thread(void) {
  work_its_updating_magic();
  // V(): signal
  V(my_sem);
}
```

### Q2
*A particular abstraction only allows a maximum of 10 threads to enter the 'room' at any point in time. Further threads attempting to enter the room have to wait at the door for another thread to exit the room*
This can be implemented with a semaphore with it's count initalised to 10
```c++
my_sem = create_semaphore(10);
```

### Q3: Conditional Variables
*Multiple threads are waiting for the same thing to happen (eg. a disk block to arrive from disk). Write pseudo-code for synchronising and walking the multiple threads waiting for the same event*
**Conditional variables!**
```c++
CV *disk_cv = create_cv();
LOCK *disk_lock = create_lock();

disk_status = NOT_READY;

void waiting_thread(void) {
  // The lock makes sure only one thread can check the disk_status so that processes don't go in limbo
  lock_acquire(disk_lock);
  while (disk_status != READY) {
    cv_wait(disk_cv, disk_lock)
  }
  lock_release(disk_lock);

  // The disk is ready
  blahblahblah();
}

void disk_thread(void) {
  // handle the disk stuff

  // once ready
  lock_acquire(disk_lock);
  disk_status = READY;

  // Broadcast to all waiting threads that the condition is satisfied and they are ready
  cv_broadcast(disk_cv, disk_lock);

  lock_release(disk_lock);
}
```

---

## Deadlock

### Q4

```c++
semaphore *mutex, *data;
 
void me() {
	P(mutex);
	/* do something */
	
	P(data);
	/* do something else */
	
	V(mutex);
	
	/* clean up */
	V(data);
}
 
void you() {
	P(data)
	P(mutex);
	
	/* do something */
	
	V(data);
	V(mutex);
}
```

Deadlock is possible with the following sequence:
1. `P(mutex)` in `me()`
2. `P(data)`in `you()`
3. `P(data)`in `me()` will block the thread since data is being used by you
4. `P(mutex)` in `you()` will also block the thread since data is being used by me

To prevent deadlock, order the acquisition of resources

### Q5 (Got this wrong)
```c++
lock *file1, *file2, *mutex;
 
void laurel() {
	lock_acquire(mutex);
	/* do something */
	
	lock_acquire(file1);
    	/* write to file 1 */
 
	lock_acquire(file2);
	/* write to file 2 */
 
	lock_release(file1);
	lock_release(mutex);
 
	/* do something */
	
	lock_acquire(file1); // CAN JUST DELETE THIS TO RESOLVE THE DEADLOCK
 
	/* read from file 1 */
	/* write to file 2 */
 
	lock_release(file2);
	lock_release(file1);
}
 
void hardy() {
    	/* do stuff */
	
	lock_acquire(file1);
	/* read from file 1 */
 
	lock_acquire(file2);
	/* write to file 2 */
	
	lock_release(file1);
	lock_release(file2);
 
	lock_acquire(mutex);
	/* do something */
	lock_acquire(file1);
	/* write to file 1 */
	lock_release(file1);
	lock_release(mutex);
}
```

Order for each thread:
`laurel`: mutex->file1->file2, (mutex and file1 released), file2->file1
`hardy`: file1->file2, mutex->file1

**Deadlock prone**

---

## Concurrency and Deadlock

### Q9
*Two processes are attempting to read independent blocks from a disk, which involves issuing a seek command and a read command. Each process is interrupted by the other in between its seek and read. When a process discovers the other process has moved the disk head, it re-issues the original seek to re-position the head for itself, which is again interrupted prior to the read. This alternate seeking continues indefinitely, with neither process able to read their data from disk.* 
- *Is this deadlock, starvation, or livelock?*
  It is livelock, because **two or more processes** are constantly changing state, but never actually progresses to completion. (Starvation is only when one process can't run)
- *How would you change the system to prevent the problem?*
  - The problem is that a process is trying to read from the other whilst it is seeking, causing a reset, and thus, livelock
  - A solution needs to prevent a seek and a read being interrupted: can be done with a lock surrounding it. 

---

# Week 4

## R3000 and Assembly 
### Q1
*What is branch delay?*
Branch delay occurs when one instruction in the 5-stage pipeline is executing whilst a previous instruction has already completed execution. This affects jump instructions, since when a jump instruction has finished executing, the instruction following the jump is still in the pipeline, and hence, will always be executed

### Q2

```c++
#include <stdio.h>

/* function protoypes, would normally be in header files */
int arg1(int a);
int arg2(int a, int b);
int arg3(int a, int b, int c);
int arg4(int a, int b, int c, int d);
int arg5(int a, int b, int c, int d, int e );
int arg6(int a, int b, int c, int d, int e, int f);

/* implementations */
int arg1(int a)
{
  return a;
}

int arg2(int a, int b)
{
  return a + b;
}  

int arg3(int a, int b, int c)
{
  return a + b + c;
}

int arg4(int a, int b, int c, int d)
{
  return a + b + c + d;
}

int arg5(int a, int b, int c, int d, int e )
{
  return a + b + c + d + e;
}

int arg6(int a, int b, int c, int d, int e, int f)
{
  return a + b + c + d + e + f;
}

/* do nothing main, so we can compile it */
int main()
{
}
```

```assembly
004000f0 <arg1>:
  4000f0:       03e00008        jr      ra
  4000f4:       00801021        move    v0,a0

004000f8 <arg2>:
  4000f8:       03e00008        jr      ra
  4000fc:       00851021        addu    v0,a0,a1

00400100 <arg3>:
  400100:       00851021        addu    v0,a0,a1
  400104:       03e00008        jr      ra
  400108:       00461021        addu    v0,v0,a2

0040010c <arg4>:
  40010c:       00852021        addu    a0,a0,a1
  400110:       00861021        addu    v0,a0,a2
  400114:       03e00008        jr      ra
  400118:       00471021        addu    v0,v0,a3

0040011c <arg5>:
  40011c:       00852021        addu    a0,a0,a1
  400120:       00863021        addu    a2,a0,a2
  400124:       00c73821        addu    a3,a2,a3
  400128:       8fa20010        lw      v0,16(sp)
  40012c:       03e00008        jr      ra
  400130:       00e21021        addu    v0,a3,v0

00400134 <arg6>:
  400134:       00852021        addu    a0,a0,a1
  400138:       00863021        addu    a2,a0,a2
  40013c:       00c73821        addu    a3,a2,a3
  400140:       8fa20010        lw      v0,16(sp)
  400144:       00000000        nop
  400148:       00e22021        addu    a0,a3,v0
  40014c:       8fa20014        lw      v0,20(sp)
  400150:       03e00008        jr      ra
  400154:       00821021        addu    v0,a0,v0

00400158 <main>:
  400158:       03e00008        jr      ra
  40015c:       00001021        move    v0,zero
```

- *What register do functions to return the return value*
  v0
- *Why is there no stack reference in arg2*
  Because we're not using any other registers or local variables and we're not calling any functions, we don't need to save the state.
- *What does jr ra do?*
  It changes the program counter to the return address: the caller
- *Which register contains the first argument of the function*
  a0
- *Why is the move function in arg1 after the jr instruction?*
  Because of branch delay, the instruction after the jr instruction will always execute. Hence, logically, it would be the same as if the move instruction was before the jr instruction
- *Why does arg5 and arg6 reference the stack?*
  Because there are only 4 argument registers, the remaining arguments would be stored onto the stack

### Q3

```c++
#include <stdio.h>
#include <unistd.h>

char teststr[] = "\nThe quick brown fox jumps of the lazy dog.\n";

void reverse_print(char *s)
{
  if (*s != '\0') {
    reverse_print(s+1);
    write(STDOUT_FILENO,s,1);
  }
}

 int main()
{
  reverse_print(teststr);
}

```

```assembly
004000f0 <reverse_print>:
  # Prologue
  # Allocate space on the stack to store variables
  4000f0:       27bdffe8        addiu   sp,sp,-24
  # Store the return address so that function can properly return to caller later
  4000f4:       afbf0014        sw      ra,20(sp)
  # Store s0: By convention this register is expected to be unchanged by function that called it
  4000f8:       afb00010        sw      s0,16(sp)

  # Load the argument into v0 (s[0])
  4000fc:       80820000        lb      v0,0(a0)
  400100:       00000000        nop
  # If first char of string is the null terminator, jump to epilogue
  400104:       10400007        beqz    v0,400124 <reverse_print+0x34>
  400108:       00808021        move    s0,a0
  40010c:       0c10003c        jal     4000f0 <reverse_print>
  400110:       24840001        addiu   a0,a0,1

  # Call write
  400114:       24040001        li      a0,1
  400118:       02002821        move    a1,s0
  40011c:       0c1000af        jal     4002bc <write>
  400120:       24060001        li      a2,1

  # Epilogue
  400124:       8fbf0014        lw      ra,20(sp)
  400128:       8fb00010        lw      s0,16(sp)
  40012c:       03e00008        jr      ra
  400130:       27bd0018        addiu   sp,sp,24
```

*What is the maximum depth the stack can grow when this function is called?*
The depth of the stack would only be limited by hardware limitations, thus stack growth is unbounded until stack overflow occurs

### Q4
*Why is recursion or large arrays of local variables avoided by kernel programmers?*
Recursion and large arrays may be dangerous since the kernel stack is a limited resources, and stack overflows may compromise the entire OS.

---

## Threads

### Q7
*A web server is constructed such that it is multithreaded. If the only way to read from a file is a normal blocking read system call, do you think user-level threads or kernel-level threads are being used for the web server? Why?*
If user-level threads were used, when a thread makes a normal blocking read system call, the entire process is blocked until the system call returns, whereas if kernel-level threads are used, the blocking of one thread would allow the scheduler to run another thread during the syscall. Hence, kernel-level threads may be more ideal.

### Q8
*Assume a multi-process operating system with single-threaded applications. The OS manages the concurrent application requests by having a thread of control within the kernel for each process. Such a OS would have an in-kernel stack assocaited with each process.*

*Switching between each process (in-kernel thread) is performed by the function switch_thread(cur_tcb,dst_tcb). What does this function do?*

- The important registers associated with the current thread would need to be stored onto its stack.
- Save the stack pointer (where we are in this particular thread's stack) to the TCB (Thread Control Block)
- Get the stack pointer to use from the destination TCB and set the stack pointer to that value
- Restore the registers from the destination stack
- Restart running the destination thread

---

## Kernel Entry and Exit

### Q9
*What is the EPC register? What is it used for?*
The Exception Program Counter is used to store the address of the instruction that caused the exception. It is used by the exception handler to to restart execution at the point where it was interrupted by the exception.

### Q10
*What happens to the KUc and IEc bits in the STATUS register when an exception occurs? Why? How are they restored?*
KUc and IEc stores the current state of the CPU: KUc stores whether the CPU is in kernel mode or user mode and IEc stores whether the CPU currently has interrupts enabled or masked. When an exception occurs, KUc and IEc will be updated and the values that were previously on those bits will be shifted so that they can be restored on return (rfe). They are restored through an rfe (restore from exception) instruction by shifting the bits back to KUc and IEc

### Q11
*What is the value of ExcCode in the Cause register immediately after a system call exception occurs?*
8

### Q12
*Why must kernel programmers be especially careful when implementing system calls?*
System calls with poor argument checking or implementation can result in a malicious or buggy program crashing, or compromising the operating system.

### Q14
*In the example given in lectures, the library function read invoked the read system call. Is it essential that both have the same name? If not, which name is important?*
No, the name of the system call is not important as system calls are typically referenced by number, so the two do not need to have the same name. The name that is important is the library function, because that is what the user will use to make the system call.

### Q15
*To a programmer, a system call looks like any other call to a library function. Is it important that a programmer know which library function result in system calls? Under what circumstances and why?*
As far as logic is concerned, using library functions that make system calls won't have any effect. However, crossing the user-kernel boundary may be expensive, so a programmer that wants maximum efficiency may want to minimize system calls and user alternative functions.

### Q16
Bruh I cannot be bothered to write this up, maybe sometime in the not so foreseeable future
