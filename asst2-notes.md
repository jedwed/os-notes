
- [Basic ASST2 Spec](#basic-asst2-spec)

---
# Basic ASST2 Spec
- Implement the following system calls: `open()`, `read()`, `write()`, `lseek()`, `close()`, `dup2()`
  - Assume you need to support `fork()`
    - Document the concurrency issues introduced by `fork()`
    - However, should not synchronise the actual code
      - Can assume will only be tested a single process at a time
    - The data structures should not need significant changes to support `fork()`
      - Except for synchronisation
  - User-level exists
    - asst2
    - C libraries
  - An existing framework and code for
    - system call dispatching
    - VFS
    - emufs
    - drivers
