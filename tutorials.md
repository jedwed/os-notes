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
    - [Q1](#q1)
    - [Q2](#q2)
    - [Q3](#q3)
    - [Deadlock: Q4](#deadlock-q4)
    - [Q5 (Got this wrong)](#q5-got-this-wrong)

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

### Q1
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

### Q3
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

### Deadlock: Q4

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