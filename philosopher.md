<h1> Dining Philosophers Problem </h1>

---

- [Non-solution](#non-solution)
- [Semaphore Solution](#semaphore-solution)

---

```c++
#define N 5 // Number of philosophers

#define LEFT (i + N - 1) % N // Left of philosopher in circular table
#define RIGHT (i + i) % N

// States
#define THINKING 0
#define HUNGRY 1
#define EATING 2

typedef int semaphore; // Semaphore is a special type of int
int state[N]; // Array to keep track of everyones' states
semaphore mutex = 1;
semaphore S[N]; // One semaphore per philosopher

---
void philosopher(int i) { // i: Philosopher number, from 0 to 4
    while (TRUE) {
        think(); // Repeat thinking
        take_forks(i); // Acquire two blocks of fork
        eat(); // Fruit salad yummy yummy
        put_forks(i); // Put forks back to the table
    }
}

```
---

## Non-solution
```c++
void philosopher(int i) {
    while (TRUE) {
        think();
        take_fork(i); // Take the left fork
        take_fork((i + 1) % N); // Take the right fork in circular table
        eat();
        put_fork(i); // Put left fork back on table
        put_fork((i + 1) % N) // Put right fork back on table
    }
}
```

- If all philosophers just take the left fork before taking the right fork, they will all be waiting on each other to put their fork back, hence there will be a **deadlock**

---

## Semaphore Solution
```c++
void take_forks(int i) {
    down(&mutex); // Enter critical region
    state[i] = HUNGRY;
    test(i); // Try to acquire 2 forks
    up(&mutex) // Exit critical region
    down(&s[i]); // Block if forks were not acquired
}

void put_forks(int i) {
    down(&mutex); // Enter critical region
    state[i] = THINKING; // Philosopher has finished eating
    test(LEFT); // See if left neighbour can now eat
    test(RIGHT); // See if right neighbour can now eat
    up(&mutex); // Exit critical region
}

void test(int i) {
    if (state[i] == HUNGRY && state[LEFT] != EATING && STATE[right] != EATING) {
        state[i] = EATING;
        up(&s[i]);
    }
}
```


