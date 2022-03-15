<h1>Thread API</h1>


---

- [Subset of POSIX threads API](#subset-of-posix-threads-api)

---

## Subset of POSIX threads API
```c++
int pthread_create(pthread_t *, const pthread_attr_t *,
void *(*)(void *), void *);
void pthread_exit(void *);
int pthread_mutex_init(pthread_mutex_t *, const pthread_mutexattr_t *);int pthread_mutex_destroy(pthread_mutex_t *);
int pthread_mutex_lock(pthread_mutex_t *);
int pthread_mutex_unlock(pthread_mutex_t *);
int pthread_rwlock_init(pthread_rwlock_t *,
const pthread_rwlockattr_t *);
int pthread_rwlock_destroy(pthread_rwlock_t *);
int pthread_rwlock_rdlock(pthread_rwlock_t *);
int pthread_rwlock_wrlock(pthread_rwlock_t *);
int pthread_rwlock_unlock(pthread_rwlock_t *);
```

