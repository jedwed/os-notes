<h1> OS/161 Synchronisation Primitives </h1>

---

- [Locks](#locks)
  - [Example usage of locks](#example-usage-of-locks)
- [Semaphores](#semaphores)
  - [Example usage of semaphores](#example-usage-of-semaphores)
- [Conditional Variables](#conditional-variables)
  - [Condition Variables and Bounded Buffers](#condition-variables-and-bounded-buffers)
    - [Non-solution](#non-solution)
    - [Solution](#solution)
  - [Alternative solution to Producer-Consumer Problem with OS/161 CVs](#alternative-solution-to-producer-consumer-problem-with-os161-cvs)

---

## Locks
- Create and destroy locks
```c++
struct lock *lock_create(const char *name); // Name solely for OS/161 debugging purposes
void lock_destroy(struct lock *)
```

- Acquire and release locks
```c++
void lock_acquire(struct lock *);
void lock_release(struct lock *);
```

### Example usage of locks
```c++
int count;
struct lock *count_lock;

main () {
    count = 0;
    count_lock = lock_create("count_lock");
    if (count_lock == NULL) 
        panic("I'm dead");
    stuffblahblah();
}

procedure inc() {
    // Sometimes: kassert(lock != NULL);
    lock_acquire(count_lock);
    count = count + 1;
    lock_release(count_lock);
}

procedure dec() {
    lock_acquire(count_lock);
    count = count - 1;
    lock_release(count_lock);
}
```

---

## Semaphores
```c++
struct semaphore *sem_create(const char *name, int intial_count);
void sem_destroy(struct semaphore *);
void P(struct semaphore *); // Wait
void V(struct semaphore *); // Signal
```

### Example usage of semaphores
```c++
int count;
struct semaphore *count_mutex;

main () {
    count = 0;
    count_mutex = sem_create("count", 1);
    if (count_mutex == NULL) 
        panic("I'm dead");
    stuffblahblah();
}

procedure inc() {
    P(count_mutex);
    count = count + 1;
    V(count_mutex);
}

procedure dec() {
    P(count_mutex);
    count = count - 1;
    V(count_mutex);
}
```

---

## Conditional Variables
```c++
// Semaphore structure doesn't exist in C, so locks are used instead to try simulate it

struct cv *cv_create(const char *name);
void cv_destroy(struct cv *);

void cv_wait(struct cv *cv, struct lock *lock);
/*
- Releases the lock and blocks
- Upon resumption, it re-acquires the lock
    - Note: We must recheck the condition we slept on
*/

void cv_signal(struct cv *cv, struct lock *lock); // Wake one thread up waiting on the conditional variable
void cv_broadcast(struct cv *cv, struct lock *lock);
/*
- Wakes one/all, does not release the lock
- First 'waiter' scheduled after signaller releases the lock will re-acquire the lock
*/
```
*Note: All three variants must hold the lock passed in*

### Condition Variables and Bounded Buffers
#### Non-solution
**Fails because it goes to sleep holding the lock**
```c++
lock_acquire(c_lock)
if (count == 0) 
    sleep();
remove_item();
count--;
lock_release(c_lock);
```

#### Solution
```c++
lock_acquire(c_lock)
while (count == 0)
    cv_wait(c_cv, c_lock); // Releases the lock to allow other processes to change the count
remove_item();
count--;
lock_release;
```

### Alternative solution to Producer-Consumer Problem with OS/161 CVs
```c++
int count = 0;
#define N 4 // Buffer size
prod() {
    while (TRUE) {
        item = produce()
        lock_acquire(1)
        while (count == N)
            cv_wait(full, 1); // Sleeps and releases the lock so that it doesn't go to sleep holding the lock, it wakes up when signalled by consumer
        insert_item(item);
        count++;
        cv_signal(empty, 1);
        lock_release(1);
    }
}

con() {
    while (TRUE) {
        lock_acquire(1)
        while (count == 0)
            cv_wait(empty, 1);
        item = remove_item();
        count--;
        cv_signal(full, 1);
        lock_release(1);
        consume(item);
    }
}
```