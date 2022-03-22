<h1>System Calls</h1>

---

- [User-Level System Call Walkthrough: Calling read()](#user-level-system-call-walkthrough-calling-read)
- [Inside the read() syscall function](#inside-the-read-syscall-function)
  - [Part 1](#part-1)
  - [Part 2](#part-2)
- [Summary](#summary)
- [System Calls: Kernel Side](#system-calls-kernel-side)
- [OS/161 Exception Handling](#os161-exception-handling)
- [Now we arrive in 'C' Kernel](#now-we-arrive-in-c-kernel)
- [What happens next?](#what-happens-next)
- [Example Program with File System Calls](#example-program-with-file-system-calls)

---

## User-Level System Call Walkthrough: Calling read()
```bash
man 2 read
```
```c++
int read(int filehandle, void *buffer, size_t size);
```
- Three arguments, one return value
- Code fragment calling the read function
- a0: filehandle, a1: pointer to buffer, a2: buffer size
```mips
400124: 02602021 move a0,s3   
400128: 27a50010 addiu a1,sp,16
40012c: 0c1001a3 jal 40068c <read>
400130: 24060400 li a2,1024
400134: 00408021 move s0,v0         # Return value from read(): Number of bytes read, -1 if fails
400138: 1a000016 blez s0,400194     # Test if return value is less than or equal to 0
<docat+0x94>
```
- Args are loaded, return value is tested

## Inside the read() syscall function
### Part 1
```mips
0040068c <read>:
  40068c: 08100190 j 400640
  <__syscall>
  400690: 24020005 li v0,5
```
- Appropiate registers are preserved
  - Arguments (a0-a3), return address (ra), tec.
  - The syscall number (5) for read is loaded in v0 (in branch delay slot)
  - Jump (*Not jump and link) to the common syscall routine 

### Part 2
```mips
00400640 <__syscall>:
  400640: 0000000c syscall # Generate a syscall exception                 
  400644: 10e00005 beqz a3,40065c <__syscall+0x1c> # If system call was successful, jump to the address
  400648: 00000000 nop
  40064c: 3c011000 lui at,0x1000
  400650: ac220000 sw v0,0(at) # errno stored
  400654: 2403ffff li v1,-1
  400658: 2402ffff li v0,-1 # Indicate error occured
  40065c: 03e00008 jr ra
  400660: 00000000 nop
```

## Summary
- From caller's perspective, the read() system call behaves like a normal function call
  - Preserves the calling convention of the language
- However, the actual function implements its own convention by agreement with the kernel
  - OS/161 example assumes the kernel preserves appropiate registers(s0-s8, sp, gp, ra)
- Most languages have similar *libraries* that interface with the operating system

## System Calls: Kernel Side
- Things left to do:
  -  Change to kernel stack
  -  Preserve registers by saving to memory(on the kernel stack)
  -  Leave saved registers somewhere accessible to
       -  Read arguments
       -  Store return values
  - Do the "read()"
  - Restore registers
  - Switch back to user stack
  - Return to application

## OS/161 Exception Handling
(Code from uniprocessor variant os OS161)
- Registers k0 and k1 are reserved for use by OS
```mips
exception:
  move k1, sp /* Save previous stack pointer in k1 */
  mfc0 k0, c0_status /* Get status register */
  andi k0, k0, CST_Kup /* Check the we-were-in-user-mode bit */
  beq k0, $0, 1f /* If clear, from kernel, already have stack */
  nop /* delay slot */

  /* Coming from user mode - load kernel stack into sp */
  la k0, curkstack /* get address of "curkstack" */
  lw sp, 0(k0) /* get its value */
  nop

1:
  mfc0 k0, c0_cause /* Now, load the exception cause. */
  j common_exception /* Skip to common code */
  nop

common_exception:
  /*
  * At this point:
  * Interrupts are off. (The processor did this for us.)* k0 contains the exception cause value.
  * k1 contains the old stack pointer.
  * sp points into the kernel stack.
  * All other registers are untouched.
  */
  /*
  * Allocate stack space for 37 words to hold the trap frame,* plus four more words for a minimal argument block.
  */
  addi sp, sp, -164

  /* The order here must match mips/include/trapframe.h. */

  sw ra, 160(sp) /* dummy for gdb */
  sw s8, 156(sp) /* save s8 */
  sw sp, 152(sp) /* dummy for gdb */
  sw gp, 148(sp) /* save gp */
  sw k1, 144(sp) /* dummy for gdb */
  sw k0, 140(sp) /* dummy for gdb */

  sw k1, 152(sp) /* real saved sp */
  nop /* delay slot for store */

  mfc0 k1, c0_epc /* Copr.0 reg 13 == PC for exception */sw k1, 160(sp) /* real saved PC */

  sw t9, 136(sp)
  sw t8, 132(sp)
  sw s7, 128(sp)
  sw s6, 124(sp)
  sw s5, 120(sp)
  sw s4, 116(sp)
  sw s3, 112(sp)
  sw s2, 108(sp)
  sw s1, 104(sp)
  sw s0, 100(sp)
  sw t7, 96(sp)
  sw t6, 92(sp)
  sw t5, 88(sp)
  sw t4, 84(sp)
  sw t3, 80(sp)
  sw t2, 76(sp)
  sw t1, 72(sp)
  sw t0, 68(sp)
  sw a3, 64(sp)
  sw a2, 60(sp)
  sw a1, 56(sp)
  sw a0, 52(sp)
  sw v1, 48(sp)
  sw v0, 44(sp)
  sw AT, 40(sp)
  sw ra, 36(sp)

  /*
  * Save special registers.
  */
  mfhi t0
  mflo t1
  sw t0, 32(sp)
  sw t1, 28(sp)
  /*
  * Save remaining exception context information.
  */
  sw k0, 24(sp) /* k0 was loaded with cause earlier */mfc0 t1, c0_status /* Copr.0 reg 11 == status */sw t1, 20(sp)
  mfc0 t2, c0_vaddr /* Copr.0 reg 8 == faulting vaddr */sw t2, 16(sp)
  /*
  * Pretend to save $0 for gdb's benefit.
  */
  sw $0, 12(sp)

  /*
  * Prepare to call mips_trap(struct trapframe *)
  */
  addiu a0, sp, 16 /* set argument */
  jal mips_trap /* call it */
  nop /* delay slot */
```

```c++
struct trapframe {
  u_int32_t tf_vaddr; /* vaddr register */
  u_int32_t tf_status; /* status register */
  u_int32_t tf_cause; /* cause register */
  u_int32_t tf_lo;
  u_int32_t tf_hi;
  u_int32_t tf_ra; /* Saved register 31 */
  u_int32_t tf_at; /* Saved register 1 (AT) */
  u_int32_t tf_v0; /* Saved register 2 (v0) */
  u_int32_t tf_v1; /* etc. */
  u_int32_t tf_a0;
  u_int32_t tf_a1;
  u_int32_t tf_a2;
  u_int32_t tf_a3;
  u_int32_t tf_t0;
  ⁞
  u_int32_t tf_t7;
  u_int32_t tf_s0;
  ⁞
  u_int32_t tf_s7;
  u_int32_t tf_t8;
  u_int32_t tf_t9;
  u_int32_t tf_k0; /* dummy (see exception.S comments)
  */
  u_int32_t tf_k1; /* dummy */
  u_int32_t tf_gp;
  u_int32_t tf_sp;
  u_int32_t tf_s8;
  u_int32_t tf_epc; /* coprocessor 0 epc register */
};
```

|        |
| ------ |
| epc    |
| s8     |
| sp     |
| gp     |
| k1     |
| k0     |
| t9     |
| t8     |
| ...    |
| at     |
| ra     |
| hi     |
| lo     |
| cause  |
| status |
| vaddr  |
- By creating a pointer to 'vaddr' of type `struct trapframe *`, we can access the user's saved registers as normal variables within 'C'

## Now we arrive in 'C' Kernel
```c++
/*
* General trap (exception) handling function for mips.
* This is called by the assembly-language exception handler once
* the trapframe has been set up.
*/
void
mips_trap(struct trapframe *tf)
{
  u_int32_t code, isutlb, iskern;
  int savespl;

  /* The trap frame is supposed to be 37 registers long. */
  assert(sizeof(struct trapframe)==(37*4));

  /* Save the value of curspl, which belongs to the old context. */
  savespl = curspl;

  /* Right now, interrupts should be off. */
  curspl = SPL_HIGH;
```

## What happens next?
-The kernel deals with whatever caused the exception (syscall, interrupt, page fault, may modify the trapframe, eg. store return in v0, zero in a3)
- mips_trap eventually returns

```mips
exception_return:
  /* 16(sp) no need to restore tf_vaddr */
  lw t0, 20(sp) /* load status register value into t0 */
  nop /* load delay slot */
  mtc0 t0, c0_status /* store it back to coprocessor 0 */
  /* 24(sp) no need to restore tf_cause */

  /* restore special registers */
  lw t1, 28(sp)
  lw t0, 32(sp)
  mtlo t1
  mthi t0

  /* load the general registers */
  lw ra, 36(sp)

  lw AT, 40(sp)
  lw v0, 44(sp)
  lw v1, 48(sp)
  lw a0, 52(sp)
  lw a1, 56(sp)
  lw a2, 60(sp)
  lw a3, 64(sp)

  lw t0, 68(sp)
  lw t1, 72(sp)
  lw t2, 76(sp)
  lw t3, 80(sp)
  lw t4, 84(sp)
  lw t5, 88(sp)
  lw t6, 92(sp)
  lw t7, 96(sp)
  lw s0, 100(sp)
  lw s1, 104(sp)
  lw s2, 108(sp)
  lw s3, 112(sp)
  lw s4, 116(sp)
  lw s5, 120(sp)
  lw s6, 124(sp)
  lw s7, 128(sp)
  lw t8, 132(sp)
  lw t9, 136(sp)

  /* 140(sp) "saved" k0 was dummy garbage anyway */
  /* 144(sp) "saved" k1 was dummy garbage anyway */

  lw gp, 148(sp) /* restore gp */
  /* 152(sp) stack pointer - below */
  lw s8, 156(sp) /* restore s8 */
  lw k0, 160(sp) /* fetch exception return PC into k0 */
  
  lw sp, 152(sp) /* fetch saved sp (must be last) */
  
  /* done */
  jr k0 /* jump back */
  rfe /* in delay slot */
  .end common_exception
```

---

## Example Program with File System Calls

```c++
/* File copy program. Minimal error checking and reporting */

#include <sys/types.h>
#include <fcntl.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]);               /* ANSI prototype */

#define BUF_SIZE 4096
#define OUTPUT_MODE 0700                        /* Protection bits for output file */

int main(int argc, char *argv[]) {
  int in_fd, out_fd, rd_count, wt_count;
  char buffer[BUF_SIZE];

  if (argc != 3) exit(1);

  /* Open the input file and create the output file */
  in_fd = open(argv[1], O_RDONLY)               /* Open the source file */
  if (in_fd < 0) exit(2);                       /* If it cannot be opened, exit */
  out_fd = open(argv[2], OUTPUT_MODE)           /* Create the destination file */
  if (out_fd < 0) exit(3);                      /* If it cannot be created, exit */

  /* Copy loop */
  while (TRUE) {
    rd_count = read(in_fd, buffer, BUF_SIZE);   /* Read a block of data */
    if (rd_count <= 0) break;                   /* If end of file or error, exit loop */
    wt_count = write(out_fd, buffer, rd_count)  /* Write data */
    if (wt_count <= 0) exit(4);                 /* Exit if error */

    /* Close the files */
    close(in_fd);
    close(out_fd);
    if (rd_count == 0)                          /* No error on last read */
      exit(0);
    else                                        /* Error on last read */
      exit(5);
  }
}

```