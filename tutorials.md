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
- [Week 5](#week-5)
  - [Memory Hierachy and Caching](#memory-hierachy-and-caching)
    - [Q1](#q1-1)
    - [Q2](#q2-2)
  - [Files and file systems](#files-and-file-systems)
    - [Q4](#q4-2)
    - [Q5](#q5)
    - [Q7](#q7-1)
    - [Q8](#q8-1)
    - [Q9](#q9-2)
    - [Q10](#q10-1)
    - [Q11](#q11-1)
    - [Q12](#q12-1)
- [Week 7](#week-7)
  - [Files and File Systems](#files-and-file-systems-1)
    - [Q1](#q1-2)
    - [Q2](#q2-3)
    - [Q3](#q3-1)
    - [Q4](#q4-3)
    - [Q5](#q5-1)
    - [Q6](#q6)
    - [Q7](#q7-2)
    - [Q8](#q8-2)
    - [Q9](#q9-3)
    - [Q10](#q10-2)
- [Week 8](#week-8)
  - [Memory Management](#memory-management)
    - [Q1](#q1-3)
    - [Q2](#q2-4)
    - [Q3](#q3-2)
    - [Q4](#q4-4)
    - [Q5](#q5-2)
    - [Q6](#q6-1)
  - [Virtual Memory](#virtual-memory)
    - [Q7](#q7-3)
    - [Q8](#q8-3)
    - [Q9](#q9-4)
    - [Q10](#q10-3)
    - [Q11](#q11-2)
    - [Q12](#q12-2)
    - [Q13](#q13)
    - [Q14](#q14-1)
    - [Q17](#q17)
- [Week 10](#week-10)
  - [Virtual Memory](#virtual-memory-1)
    - [Q1](#q1-4)
    - [Q2](#q2-5)
    - [Q3](#q3-3)
    - [Q4](#q4-5)
  - [Multi-processors](#multi-processors)
    - [Q5](#q5-3)
    - [Q6](#q6-2)
    - [Q7](#q7-4)
    - [Q8](#q8-4)
  - [Scheduling](#scheduling)
    - [Q9](#q9-5)
    - [Q10](#q10-4)
    - [Q11](#q11-3)
    - [Q12](#q12-3)
    - [Q13](#q13-1)

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

```mips
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

```mips
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

---

# Week 5

## Memory Hierachy and Caching

### Q1
*Describe the memory hierarchy. What types of memory appear in it? What **are** the characteristics of the memory as one moves through the hierarchy? How can do memory hierarchies provide both fast access times and large capacity?*
- The memory hierachy separates the hardware storage based on their speed. As one moves down the hierachy, the storage devices will decrease in cost per bit, increase in capacity and increase in access time.
(Also decreasing in volatility)

![](https://media.geeksforgeeks.org/wp-content/uploads/Untitled-drawing-4-4.png)

### Q2
*Given that disks can stream data quite fast (1 block in tens of microseconds), why are average access times for a block in milliseconds?*
- It needs to search for the correct block: affected by rpm (rotation per minute) speed.

---

## Files and file systems

### Q4
*Consider a file currently consisting of 100 records of 400 bytes. The filesystem uses fixed blocking, i.e. one 400 byte record is stored per 512 byte block. Assume that the file control block (and the index block, in the case of indexed allocation) is already in memory. Calculate how many disk I/O operations are required for contiguous, linked, and indexed (single-level) allocation strategies, if, for one record, the following conditions hold. In the contiguous-allocation case, assume that there is no room to grow at the beginning, but there is room to grow at the end of the file. Assume that the record information to be added is stored in memory.*

- *The record is added at the beginning.*
  **Contiguous**: 100 reads + 100 writes (shifting all blocks to make space) + 1 write
  **Linked**: 1 write
  **Indexed**: 1 write
- *The record is added in the middle.*
  **Contiguous**: 50 reads + 50 writes + 1 write
  **Linked**: 50 reads + 1 write
  **Indexed**: 1 write
- *The record is added at the end.*
  **Contiguous**: 1 write
  **Linked**: 50 reads + 1 write (connecting new record to the subsequent one) + 1 write (connecting the previous record to the new one)
  **Indexed**: 1 write
- *The record is removed from the end.*
  **Contiguous**: 0 write (simply write to the file control block to mark it as free, we don't care about that)
  **Linked**: 99/100 reads + 1 write
  **Indexed**: 0 writes (similar to the contiguous case)

**Notes**:
- Fixed blocking 

### Q5
*In the previous example, only 400 bytes is stored in each 512 byte block. Is this wasted space due to internal or external fragmentation?*
Internal, it's space wasted within each block itself, rather than space wasted due to unused blocks

### Q7
*Given a file which varies in size from 4KiB to 4MiB, which of the three allocation schemes (contiguous, linked-list, or i-node based) would be suitable to store such a file? If the file is access randomly, how would that influence the suitability of the three schemes?*
- Not contiguous: it will lead to a lot of internal/external framgnetation due to varying file sizes
- Not a linked list: because file access is random, using a linked list would take longer to follow the blocks (Note: for reading, random access file is very quick)
- Thus, i-node based allocation would be ideal (it is the most popular nowadays after all)

### Q8
*Why is there VFS Layer in Unix?*
Because most modern systems require the ability to support many different file systems. Hence, a VFS provides a single interface, simplifying interation with various file systems

### Q9
*How does choice of block size affect file system performance. You should consider both sequential and random access.*
Large blocks may cause more internal fragmentation: wasted space since blocks may not be required to be that large.
Small blocks may cause more external fragmentation and there may be much more data you may need to store in each block. Also, there may be more blocks to read from, slowing down performance

### Q10
*Is the open() system call in UNIX essential? What would be the consequence of not having it?*
We could use read() and write() automatically open the file, but it would slow down performance

### Q11
*Some operating system provide a rename system call to give a file a new name. What would be different compared to the approach of simply copying the file to a new name and then deleting the original file?*
The i-node is kept, and it simply creates a new directory entry pointing to the same inode

### Q12
*In both UNIX and Windows, random file access is performed by having a special system call that moves the current position in the file so the subsequent read or write is performed from the new position. What would be the consequence of not having such a call. How could random access be supported by alternative means?*
Without being able to move the file pointer, random access is either extremely inefficient as one would have to read sequentially from the start each time until the appropriate offset is arrived at, or the an extra argument would need to be added to read or write to specify the offset for each operation.


---

# Week 7

## Files and File Systems

### Q1
*Why does Linux pre-allocate up to 8 blocks on a write to a file.*
It makes sequential accesses to file quicker since they are contiguous, better locality

### Q2
*Linux uses a buffer cache to improve performance. What is the drawback of such a cache? In what scenario is it problematic? What alternative would be more appropriate where a buffer cache is inappropriate?*
- Data being written to the buffer cache may be lost if it hasn't been flushed to the actual disk drive.
- The alternative is using a write-through cache, where reads are written immediately to the disk. Write performance may be decreased.

### Q3
*What is the structure of the contents of a directory? Does it contain attributes such as creation times of files? If not, where might this information be stored?*
A directory is stored as a regular file that containe a list of all its entries, each entry referring to an i-node containing file name, attributes and file-inode number

### Q4

*The Unix inode structure contains a reference count. What is the reference count for? Why can't we just remove the inode without checking the reference count when a file is deleted?*

Inodes contain a reference count due to hard links. The reference count is equal to the number of directory entries that reference the inode. For hard-linked files, multiple directory entries reference a single inode. The inode must not be removed until no directory entries are left (ie, the reference count is 0) to ensure that the filesystem remains consistent.

### Q5
*Inode-based filesystems typically divide a file system partition into block groups. Each block group consists of a number of contiguous physical disk blocks. Inodes for a given block group are stored in the same physical location as the block groups. What are the advantages of this scheme? Are they any disadvantages?*

- Block groups store the inode tables and the data blocks close to each other, and the proximity leads to better performance due to reduced seek times.
- Because there are more different blocks, there may be more redundant information, eg. the super block and the group descriptors

### Q6
*Assume an inode with 10 direct blocks, as well as single, double and triple indirect block pointers. Taking into account creation and accounting of the indirect blocks themselves, what is the largest possible number of block reads and writes in order to:*
1. Read one byte
   1 access to single direct block, 1 access to double indirect block and 1 access to triple indirect block, 1 read of block: 4 reads
2. Write one byte
    - 4 writes: create single indirect block, create double indirect block, create triple indirect block, write data block.
    - 3 reads, 2 writes: read single indirect, read double indirect, read triple indirect, write triple indirect, write data block
    - Other combinations are possible

*Assume i-node is cached in memory*

### Q7
*Assume you have an inode-based filesystem. The filesystem has 512 byte blocks. Each inode has 10 direct, 1 single indirect, 1 double indirect, and 1 triple indirect block pointer. Block pointers are 4 bytes each. Assume the inode and any block free list is always in memory. Blocks are not cached.*
1. What is the maximum file size that can be stored before
  - the single indirect pointer is needed?
    There are 10 direct blocks, 5120 bytes = 5Kb
  - the double indirect pointer is needed?
    1 single indirect block - 512 bytes, each pointer is 4 bytes each. Hence 128 pointers possible
    (10 + 128) * 512 bytes = 138 * 512 = around 70KB
  - the triple indirect pointer is needed?
    (10 + 128 + 128 ^ 2) * 512 = 8MB
2. What is the maximum file size supported?
   (10 + 128 + 128 ^ 2 + 128 ^ 3) * 512 = 1056837K

### Q8
*A typical UNIX inode stores both the file's size and the number of blocks currently used to store the file. Why store both? Should not blocks = size / block size?*
Blocks used to store the file are only indirectly related to file size.

- The blocks used to store a file includes and indirect blocks used by the filesystem to keep track of the file data blocks themselves.
- File systems only store blocks that actually contain file data. Sparsely populated files can have large regions that are unused within a file.

### Q9
*How can deleting a file leave a inode-based file system (like ext2fs in Linux) inconsistent in the presence of a power failure.*
Deleting a file consists of three separate modifications to the disk:

- Mark disk blocks as free.
- Remove the directory entry.
- Mark the i-node as free.
If the system only completes a subset of the operations (due to power failures or the like), the file system is no longer consistent. See lecture slide for example of things that can go wrong.



### Q10
*How does adding journalling to a file system avoid corruption in the presence of unexpected power failures.*
Simply speaking, adding a journal addresses the issue by grouping file system updates into transactions that should either completely fail or succeed. These transactions are logged prior to manipulating the file system. In the presence of failure the transaction can be completed by replaying the updates remaining in the log.


---

# Week 8

## Memory Management

### Q1
*Describe internal and external fragmentation*
- Internal fragmentation: when there is wasted space within a block since the block size if fixed and is not completely filled. 
- External framentation: When there are gaps between blocks, thus allocating space may for an item not be possible even though there is enough space for the item.

### Q2
*What are the problems with multiprogrammed systems with fixed-partitioning?*
Using fixed-partitioning will result in poor memory utilization due to internal fragmentation. Also, processes that require memory greater than the fixed-size won't be able to run, even if there is enough memory to satisfy it.

### Q3
*Assume a system protected with base-limit registers. What are the advantages and problems with such a protected system (compared to either a unprotected system or a paged VM system)?*
- Advantages: It supports multi-processing: allowing more than one process to run at once. Because of both the base and limit, the processes will not interfere with each other.
- Disadvantages: Since memory allocation must still be contiguous and the entire process must be in memory, it is prone to external fragmentation. Also sharing memory/address spaces is not supported.

### Q4
*A program is to run on a multiprogrammed machine. Describe at which points in time during program development to execution time where addresses within the program can be bound to the actual physical memory it uses for execution? What are the implication of using each of the three binding times?*
- Compile-time: The exact address must be known in advanced, and recompiled if it ever changes. Otherwise, processes will interfere with each other. Furthermore, running the same program twice would not work, since they would be using the same physical address
- Load-time: Addresses are annotated such that when program is loaded in to physical memory, the loader can bind those addresses to the correct physical memory location. It slow loading, and there is not much flexibility
- Run-time: Special hardware translates the addresses into actual phyiscal addresses (TLB?)

### Q5
*Describe four algorithms for allocating regions of contiguous memory, and comment on their properties.*
- First-fit: The algorithm begins searching from the start of memory for the first available region within memory to assign to a process. The most intuitive, also the most commonly used
- Next-fit: Similar to first fit, except instead of beginning at the start of memory, it begins where the algorithm previously ended. The rationale behind this was to spread out memory a  cross, but it performed worse than first-fit due to greater external fragmentation
- Best-fit: The algorithm choses a block that is closest in size to the requested amount of memory, which supposedly minimized external fragmentation. However, this was much slower, since it had to serach the entire list. It also leaves a lot of small holes that are unusable
- Worst-fit: The algorithm chooses the largest block for a process. The rationale behind this was to counter the problem best-fit faced: the external fragmentation from small holes. The improvement wasn't that significant, and it still had to search through the complete list

### Q6
*What is compaction? Why would it be used?*
Compaction shifts memory contents to place all active ones in one single contiguous block to remove external fragmentation

---

## Virtual Memory

### Q7
*What is swapping? What benefit might it provide? What is the main limitation of swapping?*
Swapping is the act of relocating processes temporarily our of memory to a backing store, such that is can prioritize more important processes. The main limitation of swapping is the performance implications of transfering an entire process out of memory.

### Q8
*What is paging?*
Paging is the partitioning of a process's virtual addresses into equally sized chuncks (pages), as well as partitioning the physical memory into equally sized chunks (frames) to simplify managing address translations.

### Q9
*Why do all virtual memory system page sizes have to be a power of 2? Draw a picture.*
An address is divided into a page offset and a virtual page that is sent to the MMU to translate between a page and a frame. To make dividing the address easier, they should be an even chunk of a power of two.

### Q10
*What is the TLB? What is its function?*
The TLB stands for Translation Lookaside Buffer: it acts as a cache storing ready recently used page table entries to combat the performance issues of accessing physical memory.

### Q11
*Describe a two-level page table and how it is used to translate a virtual address into a physical address.*
durrr
```c
physical_address = (pfn << 12) | offset
```

### Q12
*Given a two-level page table (in physical memory), what is the average number of physical memory accesses per virtual memory access in the case where the TLB has a 100% miss ratio, and the case of a 95% hit ratio*
With a 100% miss ratio: you would need to one read for the first-level page table, one read for the second-level page table and one more for the actual read/write
With a 95% hit ratio: $1\times0.95 + 3 \times 0.05 = 1.1$ (1 access with 95% probability, 3 accesses with 5% probability)

### Q13
*What are the two broad categories of events causing page faults? What other event might cause page faults?*
- Accessing an invalid address (such as derefencing a NULL pointer)
- If the required pages is not resident (in disk rather than in memory)

### Q14
*Translate the following virtual addresses to Physical Addresses using the TLB. The system is a R3000. Indicate if the page is mapped, and if so if its read-only or read/write.*

*The EntryHi register currently contains 0x00000200.*

*The virtual addresses are 0x00028123, 0x0008a7eb, 0x0005cfff,0x0001c642, 0x0005b888, 0x00034000.*

*TLB*
| EntryHi    | EntryLo    |
| ---------- | ---------- |
| 0x00028200 | 0x0063f400 |
| 0x00034200 | 0x001fc600 |
| 0x0005b200 | 0x002af200 |
| 0x0008a100 | 0x00145600 |
| 0x0005c100 | 0x006a8700 |
| 0x0001c200 | 0x00a97600 |

First 5 hex digits refer to the VPN for EntryHi, and PFN for EntryLo

The 0x00028200 EntryHi Register: 0x00028 is the VPN, 0x20 is the ASID
The 0x0063f400 EntryLo Register: 0x0063f is the PFN, 0x4 in bits is 0100. - N: 0
- D: 1
- V: 0
- G: 0

- 0x00028213
20-bit address: 0x00028
Exists in TLB EntryHi: 0x00028200
However, in EntryLo, the 'valid' bit is not set, hence an invalid entry

- 0x0008a7eb
20-bit address: 0x0008a
Exists in TLB EntryHi: 0x0008a100
**Note the ASID 0x10 does not match the current address space of 0x20**
TLB EntryLo = 0x00145600
Physical address: 0x00145
0x6 in bits is 0110.
Global bit is not set, not in correct address space, invalid

- 0x0005cfff
VPN exists: 0x0005c100, **wrong address space**
However, in EntryLo (0x006a8700): 0x7 = 0111
It is dirty, it is valid and it is global: hence we can ignore the ASID and it is valid
PFN is 0x006a8
Combine with 12-bit offset: 0xfff
Final physical address: 0x006a8fff

- 0x0001c642
VPN exists, in correct address space
EntryLo: 0x00a97600
0x6 in binary is 0110: dirty bit is set and valid bit is set
Final physical address: 0x00a97642

- 0x0005b888
0x002af200
0x2: 0010, Valid
Final address: 0x002af888 read only

- 0x00034000
0x001fc600
0x6: 0110: 
0x001fc000, read/write

### Q17
*Of the three page table types covered in lectures, which ones are most appropriate for large virtual address spaces that are sparsely populated (e.g. many single pages scattered through memory)?*
The 2-level suffers from internal fragmentation of page table nodes themselves. The IPT and HPT is best as it is searched via a hash, and not based on the structure of the virtual address space.

---

# Week 10

## Virtual Memory

### Q1
*What effect does increasing the page size have?*
- It would incrase internal fragmentation and the total working set size
- It increases page fault latency due to reading more data from disk
- Reduces size of page tables as a result of fewer pages. It will also lead to fewer page faults and increases TLB iits

### Q2
*Why is demand paging generally more prevalent than pre-paging?*
- Demand paging only loads pages in response to page faults, only fetching pages it needs, whereas pre-paging loads additional pages in the hopes of less pagefaults. However, it wastes memory if the pre-fetched pages aren't required. Furthermore, it may eject pages that are in the application's working set. On top of the inefficient usage of memory, it is extremely difficult to determine which pages are favourable to load at a certain point in time, thus the issues outweigh the benfits for pre-paging.

### Q3
*Describe four replacement policies and compare them.*
1. Theoretically optimal solution: Looking ahead of time at the pages and choose to replace pages that will be used at the furthest time in the future (or not at all). However, this is impossible, as we can't predict which pages may be required in the future
2. First-in, first-out (FIFO): Tosses out the oldest page. This is a suboptimal solution as the age of the page is not related to its usage, thus it might toss out pages required very soon in the future.
3. Least Recently Used (LRU): Toss out the pages that are the least recently used, as it it more likely that that page won't be used in the future. Hence, it makes LRU closer to the most optimal solution, however it is impractical, since it requires a time stamp to be kept for every page, updated on every reference
4. Clock Page Replacement: Each frame has a reference bit that is set to one if page is in use. Each frames are arranged in a 'clock', and while scanning for a replacement victim, we move along the clock until we come across a page with a zero reference bit, zeroing all the reference bits that we pass along the way

### Q4
*What is thrashing? How can it be detected? What can be done to combat it?*
- Thrashing refers to the decrease in CPU utilization as multiprogramming increases, due to not having enough memory to accommodate for all the processes' working sets, leading to more page faults.
- It can be detected by monitoring the page fault frequency and CPU utilization
- It can be recovered by suspending processes to reduce the degree of multiprogramming, making more physical memory available, so more of a process's working set can be in memory
- 
---

## Multi-processors

### Q5
*What are the advantages and disadvantages of using a global scheduling queue over per-CPU queues? Under which circumstances would you use the one or the other? What features of a system would influence this decision?*
- A global scheduling queue means each CPU has its own CPU, as opposed to a global shared ready queue. It is easier to implement and leads to a very balanced load. However, synchronisation is required to prevent race condigions, and this significantly slows down performance due to lock contention. Furthermore, since a process may be allocated to multiple CPU, cache misses for specific CPUs are much more likely

### Q6
*When does spinning on a lock (busy waiting, as opposed to blocking on the lock, and being woken up when it's free) make sense in a multiprocessor environment?*
- Depends on the amount of time spent spinning. If it spins for longer than the overhead of context switching that blocking locks require, then it is ideal.

### Q7
*Why is preemption an issue with spinlocks?*
- Spinning wastes CPU time and indirectly consumes bus bandwidth
- If the spinlock holder is preempted whilst being held, it will cause other CPUs to spin waiting for the lock to be released, which is prolonged due to pre-emption until holder is scheduled again

### Q8
*How does a read-before-test-and-set lock work and why does it improve scalability?*
```c
while (*l == BUSY || test_and_set(l)) ;
```
- read-before-test-and-set first checks if the lock is busy without accessing the memory bus, reducing bus contention. It checks the processor's cache first instead before the TSL instruction generating bus traffic

---

## Scheduling

### Q9
*What do the terms I/O bound and CPU bound mean when used to describe a process (or thread)?*
- The time to completion of a CPU-bound process is largely determined by the amount of CPU time it receives.
- The time to completion of a I/O-bound process is largely determined by the time taken to service its I/O requests. CPU time plays little part in the completion time of I/O-bound processes.

### Q10
*What is the difference between cooperative and pre-emptive multitasking?*
- Co-operative multitasking: Any threads in the running state continue running without stopping until its complete, blocks on I/O, or voluntarily yields to allow other threads to run
- However, co-operative multitasking may allow a single process to monopolise the system
- Pre-emptive scheduling allows the OS to send timer interrupts to processes to ensure fairer scheduling

### Q11
*Consider the multilevel feedback queue scheduling algorithm used in traditional Unix systems. It is designed to favour IO bound over CPU bound processes. How is this achieved? How does it make sure that low priority, CPU bound background jobs do not suffer starvation?*
- Multilevel feedback queue scheduling algorithm constantly recomputes the priority levels for each process based on their CPU usage. Hence, CPU-bound processes will be punished and have a lower priority than IO-bound processes that don't consume as much CPU. 
- A process's CPU usage will decay overtime, so low-priority CPU-bound processes will not be permanently punished and straved

### Q12
*Why would a hypothetical OS always schedule a thread in the same address space over a thread in a different address space? Is this a good idea?*
- The benefit of remaining in the address space is that you don't have to reset the TLB, reducing some performance penalties associated with context switching
- Ultimately, it's not a good idea since it can starve threads that are in a different address space
### Q13
*Why would a round robin scheduler NOT use a very short time slice to provide good responsive application behaviour?*
- Using a short timeslice would require a more context switching, generating more overhead
  
---

