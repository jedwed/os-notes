<h1>ASST3: Providing virtual memory for applications</h1>

---

- [Overview](#overview)
- [R3000 Address Space Layout](#r3000-address-space-layout)
  - [kuseg](#kuseg)
    - [The TLB](#the-tlb)
  - [kseg0](#kseg0)
- [Week 9 Tutorial](#week-9-tutorial)

---

# Overview
- Keep track of regions & pages and provide translations between virtual pages and physical frames
- Note: more than one process can exist at one point in time: two more more independent virtual address spaces that both need to be translated to a single physical address space
  - Keep track of ranges that are valid
  - Keep track of a mapping between pages and physical frames (frame number)
  - An access to an invalid region should result in a memory fault
  - An access to a valid region that doesn't exist in the physical address space should allocate and initializing a physical frame and establishing a translation

# R3000 Address Space Layout
## kuseg
- 2 gigabytes
- TLB translated (mapping loaded from page table)
- Cacheable (depending on 'N' bit)
- user-mode and kernel-mode accessible
- Page size is 4K
- Switching processes switches the translations for kuseg
- Some regions (data, text) determines by ELF file 
  - `os161-objdump -p testbin/huge`

### The TLB
- Translates between references to pages in kuseg and physical frames
- Entry in the TLB contains page number in EntryHi and frame number of EntryLo

## kseg0
- 512 megabytes
- Translation window to physical memory is **fixed, TLB not used**
- Cacheable
- Only kernel-mode accessible
- Usually where the kernel code is placed
- `physical_address = virtual_address - 0x80000000` (-2GB): need to convert to physical address to use as frame
  - `KVADDR_TO_PADDR(vaddr)`


---

# Week 9 Tutorial
`kern/arch/mips/include/tlb.h`

