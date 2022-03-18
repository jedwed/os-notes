<h1>Context Switch</h1>

---

- [Simplified OS/161 thread_switch()](#simplified-os161-thread_switch)
- [OS/161 switchframe_switch()](#os161-switchframe_switch)

---

## Simplified OS/161 thread_switch()
```c++
static
void
thread_switch(threadstate_t newstate, struct wchan *wc)
{
    /* Switch away from current thread to next thread */
    struct thread *cur, *next;
    cur = curthread;
    do {
        /* Next thread is the thread control block dequeued from head of runqueue */
        next = threadlist_remhead(&curcpu->c_runqueue);
        if (next == NULL) {
            cpu_idle();
        }
    } while (next == NULL);

    /* do the switch (in assembler in switch.S) */
    switchframe_switch(&cur->t_context, &next->t_context);
}

```


## OS/161 switchframe_switch()
- A C function 
```assembly
switchframe_switch:
    /*
    * a0 contains the address of the switchframe pointer in the old thread.
    * a1 contains the address of the switchframe pointer in the new thread.
    *
    * The switchframe pointer is really the stack pointer. The other
    * registers get saved on the stack, namely:
    *
    *   s0-s6, s8
    *   gp, ra
    *
    * The order must match <mips/switchframe.h>.
    *
    * Note that while we'd ordinarily need to save s7 too, because we
    * use it to hold curthread saving it would interfere with the way
    * curthread is managed by thread.c. So we'll just let thread.c
    * manage it.
    */

    /* Allocate stack space for saving 10 registers. 10*4 = 40 bytes */
    addi sp, sp, -40

    /* Save the registers */
    sw ra, 36(sp)
    sw gp, 32(sp)
    sw s8, 28(sp)
    sw s6, 24(sp)
    sw s5, 20(sp)
    sw s4, 16(sp)
    sw s3, 12(sp)
    sw s2, 8(sp)
    sw s1, 4(sp)
    sw s0, 0(sp)

    /* Store the old stack pointer in the old thread */
    sw sp, 0(a0)

    /* Get the new stack pointer from the new thread */
    lw sp, 0(a1)
    nop /* delay slot for load */

    /* Now, restore the registers */
    lw s0, 0(sp)
    lw s1, 4(sp)
    lw s2, 8(sp)
    lw s3, 12(sp)
    lw s4, 16(sp)
    lw s5, 20(sp)
    lw s6, 24(sp)
    lw s8, 28(sp)
    lw gp, 32(sp)
    lw ra, 36(sp)
    nop /* delay slot for load */
    
    /* and return. */
    j ra
    addi sp, sp, 40 /* in delay slot */
```