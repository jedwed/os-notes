
- [Basic ASST2 Spec](#basic-asst2-spec)
- [Calling `open()`](#calling-open)
- [`fork()`](#fork)
- [Code Walkthrough Questions](#code-walkthrough-questions)

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

# Calling `open()`
```c++
int open(const char *filename, int flags, ...)
```
- It will be in the code section that the compiler generates
- Our job is to implement this function: it will be in the OS kernel since it needs access to the FS and devices


# `fork()`

- Only need to set up console for file descriptors 1 and 2 for the very first process that starts (`run_program`)
- In future, children will inherit those same references from the parent

---

# Code Walkthrough Questions
