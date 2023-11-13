---
layout: post
title: Ring Buffer
date: 2023-11-12
tag: [algorithm]
categories: Algorithm
---

Ring Buffer也叫Circular Buffer、Circular Queue或者Cyclic Buffer，是一个固定大小、首尾相连的数据结构。多用于缓存数据流。

## 数据结构

借用一张Wikipedia上的动图展示Ring Buffer的结构和机制。

![A 24-byte keyboard circular buffer](https://upload.wikimedia.org/wikipedia/commons/f/fd/Circular_Buffer_Animation.gif)

### 特点

从上图上可以看出Ring Buffer的元素被读取后，不需要释放，其余元素不需要移动位置。由于缓冲区大小固定，所以不需要重新申请内存。由于这个特性，也不适用于需要变动缓冲区大小的应用场合。

同时由于Ring Buffer是首尾相连的，Buffer写满后，如果没有及时读取，再次进行写操作会覆盖Buffer里的旧数据。对于某些场合，不能覆盖数据，需要判断Buffer是否已经写满。由于Buffer空的时候和满的时候，读写索引都指向相同的地址，需要通过特殊的算法来区分Buffer是空的还是满的。

### 内存中的结构

由于计算机的内存是线性的，所以Ring Buffer在计算机内存中的机构实际上就是一段联系的存储空间。

例如一个空间为8的Ring Buffer：

```
+---+---+---+---+---+---+---+---+
|   |   | A | B | C | D |   |   |
+---+---+---+---+---+---+---+---+
  ^       ^           ^       |
  |     R_IDX       W_IDX     |
  +---------------------------+
```

其中读索引是2，写索引是5。读索引和写索引以外的数据是无效的数据。

每写入一个数据后，写索引向后移动一格。当写索引超过Buffer的长度时，写索引变成0，回到Buffer的开始。读索引也是同样的操作。

可以看出，每次读出的数据都是Buffer里面最老的数据，所以满足FIFO特性。

### 空还是满

由于Ring Buffer是首尾相连的，在Buffer空和满的情况下，写索引和读索引都会指向同一个地址，如何判断是空还是满呢？

#### 镜像指示

如果Ring Buffer的大小为n，逻辑地址空间为[0...n-1]。那么扩展[n...2n-1]为镜像逻辑地址空间。

当读写指针到达n之后，就进入了镜像逻辑地址空间，当读写指针到达2n的时候，再进行折返。

这里的*指针*不是*索引*。所以要添加一个标志位标识读写指针是否到到了镜像空间。

当读写索引相等的时候，如果镜像标志一样的话，就表示Buffer是空的，否则就是满的。

```
+---+---+---+---+---+---+---+---+ - + - + - + - + - + - + - + - +
|   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   | Empty
+---+---+---+---+---+---+---+---+ - + - + - + - + - + - + - + - +
              ^
         R_P -+- W_P
```

```
+---+---+---+---+---+---+---+---+ - + - + - + - + - + - + - + - +
|   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   | Full
+---+---+---+---+---+---+---+---+ - + - + - + - + - + - + - + - +
              ^                               ^
         R_P -+                               +- W_P
```

读写指针和读写索引的换算关系：

$$
Read_{index}=Read_{point}\%Length_{buffer}
$$

$$
Write_{index}=Write_{point}\%Length_{buffer}
$$

Ring Buffer结构：

```c
typedef struct _ringbuffer {
    uint8_t *buffer_ptr;
    uint16_t size;
    uint8_t read_mirror;
    uint8_t write_mirror;
    uint16_t read_index;
    uint16_t read_index;
} ringbuffer_t;
```

设置mirror标志位：

```c
int ringbuffer_write(ringbuffer_t *rb, ...) {
    ...
    rb->write_index++;
    if (rb->write_index == rb->size) {
        rb->write_index = 0;
        rb->write_mirror = !(rb->write_mirror);
    }
    ...
}

int ringbuffer_read(ringbuffer_t *rb, ...) {
    ...
    rb->read_index++;
    if (rb->read_index == rb->size) {
        rb->read_index = 0;
        rb->read_mirror = !(rb->read_mirror);
    }
    ...
}
```

空和满的判断：

```c
#define ringbuffer_is_empty(rb) \
    ((rb->read_index == rb->write_index) && \
     (rb->read_mirror == rb->write->write_index)

#define ringbuffer_is_full(rb) \
    ((rb->read_index == rb->write_index) && \
     (rb->read_mirror != rb->write->write_index)
```

#### 镜像指示扩展

如果Ring Buffer的大小是2的n次方的，镜像指示可以简化。

读索引和写索引可以用按位与的方式进行计算：

$$
Read_{index}=Read_{point}\&(Length_{buffer}-1)
$$

$$
Write_{index}=Write_{point}\&(Length_{buffer}-1)
$$

当读写指针相差\\(2^n\\)时，Buffer是满的；相差是0时，Buffer是空的。

Ring Buffer结构：

```c
#define RING_BUFFER_SIZE    (1 << 8)

typedef struct _ring_buffer {
    unsigned char *buffer_data[RING_BUFFER_SIZE];
    unsigned int read_point;
    unsigned int write_point;
    pthread_mutex_t mutex;
} ring_buffer_t;
```

空和满的判断：

```c
#define ring_buffer_is_full() \
    ((ring_buffer->read_point ^ ring_buffer->write_point) == RING_BUFFER_SIZE)

#define ring_buffer_is_empty() \
    (ring_buffer->read_point == ring_buffer->write_point)
```


## 参考实现

### Linux Kernel中的``kfifo``

Linux Kernel中的``kfifo``就是典型的Ring Buffer结构，代码在``include/linux/kfifo.h``和``lib/kfifo.c``中实现。

``kfifo``对读操作和写操作的实现非常简洁。在进行读操作和写操作时，其充分利用了无符号整型的性质。在读写操作的时候，``in``（写指针）和``out``（读指针）都是正向增加的，当达到最大值时，产生溢出，使得从0开始，进行循环使用。``in``和``out``会保存一个固定的关系：

$$
out+Length_{buffered}=in
$$

``out``永远不会超过``in``，他们相等时，Ring Buffer就是空的。

## 参考
- [1][Wikipedia:Circular Buffer](https://en.wikipedia.org/wiki/Circular_buffer)
- [2][知乎：ring buffer，一篇文章讲透它？](https://zhuanlan.zhihu.com/p/534098236)
