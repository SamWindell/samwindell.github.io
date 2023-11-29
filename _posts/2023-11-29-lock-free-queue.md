---
title: Lock-free FIF0 Queues
layout: post
tags: programming c/c++
---

I've recently been using lock-free FIFO (first-in first-out) queues quite a lot. I thought I'd share some of what I've learnt in case it can give someone else a head-start. 

Let's start with an overview of things we might want to consider first:
- Are we just dealing with a single-producer (SP) thread passing data to a single-consumer (SC) thread? In other words, is only ever _one_ thread calling Push() and another _one_ thread calling Pop()? If so, the solution is quite simple.
- Do we have multiple-producers (MP) or multiple-consumers (MC)? Do we really need both consuming and producing to be lock-free? Would wrapping either the Push() or Pop() of a SPSC queue with a mutex be sufficient?
- This kind of data structure has some drawbacks. The most common one is that we will most likely be dealing with a fixed-size buffer: the queue will have a maximum capacity.

The simplest design that I've found for a SPSC queue is from the paper ["Correct and Efficient Bounded FIFO Queues by Nhat Minh Lê, Adrien Guatto, Albert Cohen, Antoniu Pop"](https://inria.hal.science/hal-00862450/document). It includes an implementation using C11 atomics of an improved version of an Lamport's queue (from an older paper). It also proposes a faster version called WeakRB that might be worth looking at. For now, let's just take a look at the improved Lamport's queue because it's incredibly simple. The underlying data structure is fixed-size ring-buffer.

<pre>
<code>atomic_size_t front;
size_t pfront;
atomic_size_t back;
size_t cback;
T data[SIZE];

static inline void init() {
    atomic_init(front, 0);
    atomic_init(back, 0);
    pfront = cback = 0;
}

static inline bool push(T elem) {
    size_t b, f;
    b = atomic_load_explicit(&back, memory_order_relaxed);
    f = atomic_load_explicit(&front, memory_order_acquire);
    if ((b + 1) % SIZE == f) return false;
    data[b] = elem;
    atomic_store_explicit(&back, (b + 1) % SIZE, memory_order_release);
    return true;
}

static inline bool pop(T *elem) {
    size_t b, f;
    b = atomic_load_explicit(&back, memory_order_relaxed);
    f = atomic_load_explicit(&front, memory_order_acquire);
    if (b == f) return false;
    *elem = data[b];
    atomic_store_explicit(&front, (f + 1) % SIZE, memory_order_release);
    return true;
}</code>
</pre>

It would be very simple to translate this into C++11 using std::atomic if needed: C11 and C++ use the same atomics model.

I've found this algorithm pop up a few times in the wild:
- [CLAP audio plugin helpers](https://github.com/free-audio/clap-helpers/blob/4a2e3ee4d7de38912b9375a47a0e1294075909f6/include/clap/helpers/param-queue.hh)
- [Crown game engine](https://github.com/crownengine/crown/blob/9a555758e2ddf4665efb2e120fcb8a5b9eb72859/src/core/thread/spsc_queue.inl)

If multiple-producers or multiple-consumers are needed I would suggest that the first thing to consider would be to use a mutex to ensure that only one thread can push or pop simultaneously. [Have a look at how the Crown game engine does this](https://github.com/crownengine/crown/blob/master/src/core/thread/mpsc_queue.inl). However, mutexes introduce the potential of calling into the kernal and performing a proper block operation; though it's worth remembering that modern mutex implementations can often avoid doing this - they perform atomic operations in user-space before deciding if a syscall is needed (see [glibc pthread](https://github.com/lattera/glibc/blob/master/nptl/pthread_mutex_lock.c#L168) for example). That being said, any potential for blocking might not be acceptable if there you have very strict real-time requirements. If you to go down the lock-free MPMC route, have a look at these links:
- [FreeBSD buf_rung.h](https://svnweb.freebsd.org/base/release/12.2.0/sys/sys/buf_ring.h?revision=367086&view=markup)
- [The Book of Gehn - Lock-Free Queue](https://book-of-gehn.github.io/articles/2020/03/22/Lock-Free-Queue-Part-I.html)
- [Loki lock-free queue library](https://github.com/eldipa/loki)

On a slightly unrelated note, here's a link to a neat-trick regarding read/write indexes of a ring buffer: [Juho Snellman's Weblog - I've been writing ring buffers wrong all these years](https://www.snellman.net/blog/archive/2016-12-13-ring-buffers).

And finally a note to myself mostly: try to just solve the problem at hand - what exactly is needed for your current problem? Is a generic abstraction needed or will something specific but simple suffice?
