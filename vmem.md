<h1>Virtual Memory</h1>

---

- [Example assembly routine for TLB refill](#example-assembly-routine-for-tlb-refill)

---

# Example assembly routine for TLB refill
```mips
mkc0 k1, C0_CONTEXT
mkco k0, C0_EPC
lw k1,0(k1)
nop
mtco k1, C0_ENTRYLO
nop
tlbwr
jr k0   
rfe
```