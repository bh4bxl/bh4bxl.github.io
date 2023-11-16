---
layout: post
title: 多线程互斥锁的等待的方法
date: 2023-11-16
tags: [Multithreading]
categories: [Multithreading]
---

在多线程编程中，需要等待互斥锁（Mutex）的时候，可以``usleep``函数等待，同时POSIX API还提供了``pthread_cond_timedwait``和``pthread_cond_wait``函数，可以提供更好的性能。

## ``usleep``函数

``usleep``是一个轻量级的函数，用法也非常简单，参数就是要休眠的时间，可以让当前线程休眠到指定的时间在继续运行。

在多线程应用中，如果当前线程使用``usleep``等待Mutex解锁。如果进入休眠后其他线程解锁，``usleep``还是要等到指定时间休眠完成后，当前线程才能唤醒。这会导致性能上的损失。

POSIX提供了``pthread_cond_timedwait``和``pthread_cond_wait``函数可以减少这样的损失。

## ``pthread_cond_timedwait``和``pthread_cond_wait``

### 函数原型

```c
#include <pthread.h>

int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timedwait(pthread_cond_t *cond,
    pthread_mutex_t *mutex, const struct timespec *abstime);
```

其中：

- cond：触发条件
- mutex：互斥锁
- abstime：等待

调用函数成功不超时返回0。

### 用法

线程等待与加锁：

1. 加锁：``pthread_mutex_lock(&mutex)``
2. 等待：``pthread_cond_timedwait(&cond, &mutex, &abstime)``或者``pthread_cond_wait(&cond, &mutex)``
3. 解锁：``pthread_mutex_unlock(&mutex)``

发送信号量:``pthread_cond_signal(&cond)``或者``pthread_cond_broadcast(&cond)``。

## 例子

```c
/* CELEBP22 */
#define _OPEN_THREADS
#include <pthread.h>
#include <stdio.h>
#include <time.h>
#include <unistd.h>

pthread_cond_t cond;
pthread_mutex_t mutex;

int footprint = 0;

void *thread(void *arg) {
  time_t T;

  if (pthread_mutex_lock(&mutex) != 0) {
    perror("pthread_mutex_lock() error");
    return NULL;
  }
  time(&T);
  printf("starting wait at %s", ctime(&T));
  footprint++;

  if (pthread_cond_wait(&cond, &mutex) != 0) {
    perror("pthread_cond_timedwait() error");
    return NULL;
  }
  time(&T);
  printf("wait over at %s", ctime(&T));
  return NULL;
}

int main() {
  pthread_t thid;
  time_t T;
  struct timespec t;

  if (pthread_mutex_init(&mutex, NULL) != 0) {
    perror("pthread_mutex_init() error");
    return -1;
  }

  if (pthread_cond_init(&cond, NULL) != 0) {
    perror("pthread_cond_init() error");
    return -2;
  }

  if (pthread_create(&thid, NULL, thread, NULL) != 0) {
    perror("pthread_create() error");
    return -3;
  }

  while (footprint == 0)
    sleep(1);

  puts("IPT is about ready to release the thread");
  sleep(2);

  if (pthread_cond_signal(&cond) != 0) {
    perror("pthread_cond_signal() error");
    return -4;
  }

  if (pthread_join(thid, NULL) != 0) {
    perror("pthread_join() error");
    return -5;
  }
  return 0;
}
```

输出：

```
starting wait at Thu Nov 16 22:47:24 2023
IPT is about ready to release the thread
wait over at Thu Nov 16 22:47:27 2023
```


## 参考

- [1][Opengroup: pthread_cond_wait, pthread_cond_timedwait - wait on a condition](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_cond_wait.html)
- [2][IBM: pthread_cond_wait() — Wait on a condition variable](https://www.ibm.com/docs/en/zos/2.2.0?topic=functions-pthread-cond-wait-wait-condition-variable)
