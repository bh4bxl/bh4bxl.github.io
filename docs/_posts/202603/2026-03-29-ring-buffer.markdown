---
layout: post
title: Ring Buffer
date: 2026-03-29
tag: [algorithm]
categories: Algorithm
---

## Overview

Ring Buffer (also known as Circular Buffer, Circular Queue, or Cyclic Buffer) is a **fixed-size**, **circular data structure** widely used for handling continuous data streams.

Due to its characteristics:

- No data movement required
- Constant memory allocation
- High performance and cache-friendly

It is commonly used in:

- Audio systems (ALSA / DSP)
- Device drivers (UART / USB / DMA)
- Network RX/TX queues
- Virtualization (virtio queues)

## Data Structure

A ring buffer is essentially a linear memory region with logical wrap-around.

Example: a ring buffer with size = 8

```
+---+---+---+---+---+---+---+---+
|   |   | A | B | C | D |   |   |
+---+---+---+---+---+---+---+---+
  ^       ^           ^
  |     R_IDX       W_IDX
```

- `read_index = 2`
- `write_index = 5`
- Valid data: `[A, B, C, D]`

After each write:

- `write_index++`
- When reaching the end => wrap to 0

Same applies to `read_index`.

_This guarantees FIFO (First-In-First-Out) behavior._

## The Key Problem: Full vs Empty

When:

```c
read_index == write_index
```

The buffer could be:

- **Empty**
- **Full**

We need a way to distinguish these two states.

## Solution: Mirror Bit

### Concept

We extend the logical address space:

```
[0 ... n-1] → original buffer
[n ... 2n-1] → mirror space
```

Each pointer has:

- index (position in buffer)
- mirror flag (which "round" it is in)

### Structure

```c
typedef struct _ringbuffer {
    uint8_t *buffer_ptr;
    uint16_t size;

    uint8_t read_mirror;
    uint8_t write_mirror;

    uint16_t read_index;
    uint16_t write_index;
} ringbuffer_t;
```

### Pointer Update

```c
int ringbuffer_write(ringbuffer_t *rb, ...)
{
    rb->write_index++;

    if (rb->write_index == rb->size) {
        rb->write_index = 0;
        rb->write_mirror ^= 1;
    }
}
```

```c
int ringbuffer_read(ringbuffer_t *rb, ...)
{
    rb->read_index++;

    if (rb->read_index == rb->size) {
        rb->read_index = 0;
        rb->read_mirror ^= 1;
    }
}
```

### Full / Empty Detection

```c
#define ringbuffer_is_empty(rb) \
    ((rb->read_index == rb->write_index) && \
     (rb->read_mirror == rb->write_mirror))

#define ringbuffer_is_full(rb) \
    ((rb->read_index == rb->write_index) && \
     (rb->read_mirror != rb->write_mirror))
```

### Why Mirror Bit?

Compared to the common approach:

```c
(write + 1) % size == read
```

Mirror bit:

Pros:
- No wasted buffer slot
- Full utilization of memory

Cons:

- Slightly more complex

_This approach is often used in embedded systems where memory efficiency matters._

## Optimization: Power-of-Two Size

If buffer size is `2^n`, we can avoid modulo operations:

```c
index = pointer & (size - 1);
```

Example:

```c
#define RING_BUFFER_SIZE (1 << 8)

typedef struct _ring_buffer {
    unsigned char *buffer_data[RING_BUFFER_SIZE];
    unsigned int read_point;
    unsigned int write_point;
} ring_buffer_t;
```

### Full / Empty Detection

```c
#define ring_buffer_is_empty(rb) \
    (rb->read_point == rb->write_point)

#define ring_buffer_is_full(rb) \
    ((rb->write_point - rb->read_point) == RING_BUFFER_SIZE)
```

This is a **high-performance design** used in systems like the Linux kernel.

## Reference: Linux Kernel kfifo

The Linux kernel provides a ring buffer implementation called `kfifo`:

Location:
- `include/linux/kfifo.h`
- `lib/kfifo.c`

Key idea:

- `in` (write pointer) and `out` (read pointer) always increase
- Overflow is handled naturally by unsigned integer wrap-around

Invariant:

```
out <= in
in - out <= buffer_size
```

When:

- `in == out` => empty
- `in - out == size` => full

This avoids mirror bits entirely.

## Real-World Usage

### Audio Systems (ALSA / DSP)

- PCM buffer is essentially a ring buffer
- Common issues:
  - **Underrun** => consumer too fast
  - **Overrun** => producer too fast

### Virtualization (Virtio)

- Virtqueue is based on ring buffer
- Shared memory between frontend/backend (FE/BE)
- Benefits:
  - Reduce VM exits
  - Improve throughput

### Device Drivers (USB / UART)

- Interrupt handler => writes data into ring buffer
- User space => reads data

## Common Pitfalls

### Off-by-one errors

```c
(write + 1) % size
```

Easy to get wrong.

### Concurrency issues

- Producer / consumer race conditions
- Missing synchronization

In SPSC (Single Producer Single Consumer), lock-free is possible.

### Cache coherency (important in real systems)

In DMA or shared memory scenarios:

- Need to consider:
  - cacheable vs non-cacheable memory
  - cache flush / invalidate

Especially critical in:

- Hypervisor
- Embedded systems

## Summary

| **Method** | **Pros** | *Cons* |
|------------|----------|--------|
| Reserve 1 slot | Simple | Wastes space |
| Mirror bit | Full utilization | More complex |
| Power-of-two + mask | Highest performance | Requires fixed size |

## Minimal Example

```c
int ringbuffer_write(ringbuffer_t *rb, uint8_t data)
{
    if ((rb->write + 1) % rb->size == rb->read)
        return -1; // full

    rb->buffer[rb->write] = data;
    rb->write = (rb->write + 1) % rb->size;

    return 0;
}
```
