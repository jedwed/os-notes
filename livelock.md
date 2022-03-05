## Deadlock example
```c++
void proc_A() {
    lock_acquire(&res_1);
    lock_acquire(&res_2);
    use_both_res();
    lock_release(&res_2);
    lock_release(&res_1);
}

void proc_B() {
    lock_acquire(&res_2);
    lock_acquire(&res_1);
    use_both_res();
    lock_release(&res_1);
    lock_release(&res_2);
}
```

## Livelock example
```c++
/*
try_lock(): makes a lock_acquire() attempt and
returns a value as to whether it was successful or not
rather than sleeping/blocking
*/

void proc_A() {
    lock_acquire(&res_1);
    while (try_lock(&res_2) == FAIL) {
        // If it fails to get the two locks, releases the first one and
        // try to get both of them again
        lock_release(&res_1);
        wait_fixed_time();
        lock_acquire(&res_1);
    }
    use_both_res();
    lock_releases(&res_2);
    lock_releases(&res_1);
}

void proc_B() {
    lock_acquire(&res_2);
    while (try_lock(&res_1) == FAIL) {
        lock_release(&res_2);
        wait_fixed_time();
        lock_acquire(&res_2);
    }
    use_both_res();
    lock_releases(&res_1);
    lock_releases(&res_2);
}
```