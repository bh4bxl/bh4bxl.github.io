---
layout: post
title: Thread Safety in Ring Buffer: From Lock-Free to Multi-Threaded Designs
date: 2026-03-30
tag: [algorithm]
categories: Algorithm
---

## Introduction

Ring buffer is a simple yet powerful data structure widely used in:

- audio pipelines
- device drivers
- networking stacks
- virtualization systems

While its basic implementation is straightforward, **making it thread-safe is non-trivial**.

Different concurrency models require different designs, and choosing the wrong one can lead to:

- data corruption
- race conditions
- performance degradation

This article explores how to design a thread-safe ring buffer under different scenarios.

## The Core Problem

A ring buffer typically maintains two pointers:

```
read  => consumer
write => producer
```

Without synchronization, concurrent access can cause:

- producer overwriting unread data
- consumer reading incomplete data
- inconsistent pointer states

The key question is:

_Who owns `read` and `write`, and how are they synchronized?_

## Single Producer + Single Consumer (SPSC)

This is the most common and efficient model.

### Design Principle

- Producer only updates `write`
- Consumer only updates `read`
- Both can read each other's pointer

No locks required.

Example:

```c
int rb_write(rb_t *rb, uint8_t data)
{
    if ((rb->write - rb->read) == rb->size)
        return -1; // full

    rb->buffer[rb->write & rb->mask] = data;

    // memory barrier needed here
    rb->write++;

    return 0;
}

int rb_read(rb_t *rb, uint8_t *data)
{
    if (rb->read == rb->write)
        return -1; // empty

    *data = rb->buffer[rb->read & rb->mask];

    // memory barrier needed here
    rb->read++;

    return 0;
}
```

### Memory Ordering (Critical)

On modern CPUs (especially ARM):

```
write data => update pointer
```

may be reordered as:

```
update pointer => write data
```

This causes the consumer to read invalid data.

### Solution: Memory Barriers

```c
// Producer
buffer[...] = data;
smp_wmb();   // write memory barrier
write++;

// Consumer
read = rb->read;
smp_rmb();   // read memory barrier
data = buffer[...];
```

### Where This Is Used

- ALSA audio buffers
- virtio queues
- high-performance drivers

This is the **most important pattern to master**.

## Multi-Producer / Multi-Consumer (MPMC)

When multiple threads access the buffer, Lock-free SPSC no longer works.

### Option: Mutex (Recommended for most cases)

```c
pthread_mutex_lock(&lock);

write data
update pointer

pthread_mutex_unlock(&lock);
```

#### Pros / Cons

- Simple and correct
- Performance overhead (lock contention)

#### When to Use

- General application code
- Low to medium throughput systems

## Lock-Free MPMC (Advanced)

This requires:

- atomic operations
- compare-and-swap (CAS)
- careful memory ordering

Extremely complex and error-prone.

Often not worth it unless:

- ultra-low latency required
- high-frequency trading / networking

## Interrupt Context (Embedded Systems)

Very common in driver development:

- interrupt handler -> producer
- thread -> consumer

### Problem

Interrupt can occur:

- at any time
- even during pointer update

### Solution

#### Option 1: Disable Interrupts

```c
disable_interrupts();
write data
enable_interrupts();
```

#### Option 2: Spinlock

```c
spin_lock_irqsave(&lock, flags);
...
spin_unlock_irqrestore(&lock, flags);
```

### Key Insight

Interrupt context is effectively a **preemptive concurrent producer**.

## Summary

| **Scenario** | **Solution** | **Performance** |
|--------------|--------------|-----------------|
| SPSC | Lock-free | ***** |
| MPMC | Mutex | *** |
| MPMC lock-free | Atomic + CAS | ***** (but complex) |
| Interrupt + thread | Disable IRQ / spinlock | **** |

## Practical Guidelines

### Always prefer SPSC if possible

If your system allows, redesign instead of adding locks.

### Use power-of-two size

```c
index = pointer & (size - 1);
```

Avoid modulo.

### Memory ordering matters more than locks

Most bugs are **not about locks**, but about **reordering**.

### Volatile is NOT enough

```c
volatile int write;
```

Does not guarantee correctness.

## Real-World Insight

In real systems:

- audio pipelines => SPSC
- virtio => SPSC + shared memory
- drivers => interrupt + SPSC

Most high-performance designs avoid locks entirely.

## Final Thoughts

Thread safety in ring buffers is not just about adding locks.

It is about:

- ownership of data
- memory visibility
- choosing the right concurrency model

A correct design is often simpler than a heavily synchronized one.

## Appendix: Rust Implementation of Ring Buffer

This Rust version is not intended to replace low-level C implementations, but to demonstrate how the same design can be expressed with stronger safety guarantees.

Below are two typical implementations:

- A **lock-free SPSC (Single Producer Single Consumer)** ring buffer
- A **mutex-protected** version for general multi-threaded use

### Lock-Free SPSC Ring Buffer

This version is suitable for:

- audio pipelines
- driver-level data paths
- virtio-style queues

It relies on:

- atomic operations
- memory ordering (Acquire / Release)

```rust
use std::cell::UnsafeCell;
use std::sync::atomic::{AtomicUsize, Ordering};

pub struct SpscRingBuffer<T> {
    buffer: Vec<UnsafeCell<Option<T>>>,
    mask: usize,
    read: AtomicUsize,
    write: AtomicUsize,
}

unsafe impl<T: Send> Send for SpscRingBuffer<T> {}
unsafe impl<T: Send> Sync for SpscRingBuffer<T> {}

impl<T> SpscRingBuffer<T> {
    pub fn new(size: usize) -> Self {
        assert!(size.is_power_of_two(), "size must be a power of two");

        let mut buffer = Vec::with_capacity(size);
        for _ in 0..size {
            buffer.push(UnsafeCell::new(None));
        }

        Self {
            buffer,
            mask: size - 1,
            read: AtomicUsize::new(0),
            write: AtomicUsize::new(0),
        }
    }

    pub fn push(&self, value: T) -> Result<(), T> {
        let read = self.read.load(Ordering::Acquire);
        let write = self.write.load(Ordering::Relaxed);

        if (write - read) == self.buffer.len() {
            return Err(value);
        }

        let index = write & self.mask;

        unsafe {
            *self.buffer[index].get() = Some(value);
        }

        self.write.store(write + 1, Ordering::Release);
        Ok(())
    }

    pub fn pop(&self) -> Option<T> {
        let read = self.read.load(Ordering::Relaxed);
        let write = self.write.load(Ordering::Acquire);

        if read == write {
            return None;
        }

        let index = read & self.mask;

        let value = unsafe { (*self.buffer[index].get()).take() };

        self.read.store(read + 1, Ordering::Release);
        value
    }
}
```

Notes:

- `UnsafeCell` is required because Rust normally forbids mutable aliasing
- Safety is guaranteed by the **SPSC access model**
- `Acquire/Release` ensures proper memory visibility between producer and consumer

### Mutex-Protected Ring Buffer

This version is simpler and works for:

- multiple producers
- multiple consumers

```rust
use std::sync::Mutex;

pub struct MutexRingBuffer<T> {
    inner: Mutex<Inner<T>>,
}

struct Inner<T> {
    buffer: Vec<Option<T>>,
    read: usize,
    write: usize,
    len: usize,
    capacity: usize,
}

impl<T> MutexRingBuffer<T> {
    pub fn new(capacity: usize) -> Self {
        let mut buffer = Vec::with_capacity(capacity);
        for _ in 0..capacity {
            buffer.push(None);
        }

        Self {
            inner: Mutex::new(Inner {
                buffer,
                read: 0,
                write: 0,
                len: 0,
                capacity,
            }),
        }
    }

    pub fn push(&self, value: T) -> Result<(), T> {
        let mut inner = self.inner.lock().unwrap();

        if inner.len == inner.capacity {
            return Err(value);
        }

        inner.buffer[inner.write] = Some(value);
        inner.write = (inner.write + 1) % inner.capacity;
        inner.len += 1;

        Ok(())
    }

    pub fn pop(&self) -> Option<T> {
        let mut inner = self.inner.lock().unwrap();

        if inner.len == 0 {
            return None;
        }

        let value = inner.buffer[inner.read].take();
        inner.read = (inner.read + 1) % inner.capacity;
        inner.len -= 1;

        value
    }
}
```

While Rust provides stronger safety guarantees compared to C, the **core design principles of ring buffers remain the same**:

- circular indexing
- separation of read/write ownership
- careful handling of full/empty conditions
- memory ordering in concurrent scenarios

In performance-critical systems (e.g., audio, drivers, virtualization), the **SPSC lock-free design** is still the preferred choice.
