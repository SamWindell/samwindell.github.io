---
title: Atomic lock-free linked list
layout: post
tags: programming c/c++
---

For [Mirage](https://frozenplain.com/mirage), I need to load sample-library configuration files from disk, including lots of audio files which need decoding into memory for other threads to use. A [new update](https://frozenplain.com/code-your-own-libraries-devlog-5/) that I'm working on makes this a little trickier because it introduces the requirement that the set of audio files can change on the fly; changes to the Lua configuration script should cause Mirage to automatically apply them.

This is a problem of sharing memory across threads. The requirements are as follows:
- Communication is between exactly 2 threads: the reading thread, and the writing thread.
- The reading thread (the GUI thread) should be able access and iterate the available sample-libraries and audio files with absolute minimum of overhead.
- The writing thread (the thread that reads files from disk) should be able to modify or remove items from the sample-library list. And it should be able to periodically delete unreferenced items.

I think the data structure that I've come up with to solve this fiddly problem is pretty neat. It combines 3 well-known patterns: singly-linked lists (3 of them), a memory-arena and weak reference counting. It makes extensive use of atomic operations instead of locks. See the whole code [here](https://gist.github.com/SamWindell/5f9eb5226eeb110f91e3917015e98e8e).

Let's start with what the struct looks like:

```cpp
template <typename ValueType>
struct AtomicRefList {
    Atomic<Node *> live_list {}; // reader-thread or writer-thread
    Node *dead_list {}; // writer-thread
    Node *free_list {}; // writer-thread
    ArenaAllocator arena {PageAllocator::Instance()}; // writer-thread
};
```

Above we can see a few different types. The `Node` type is what we use to store items in the list. We shall cover that below. The `Atomic` type is just a wrapper around clang's builtin atomics such as `__atomic_fetch_add`. You can simply replace it with `std::atomic`.

We also have an `ArenaAllocator`. I'm not going to include the implementation of the `ArenaAllocator` here because things will get a bit complicated. Essentially it can be used to allocate memory that is mostly contiguous and can be freed all at once. When memory is requested, it tries to bump-allocate the requested amount by incrementing a cursor on a large memory region. If there's no room in its current region, it will create a new region and return memory from that instead. The important things to note here are: memory never moves, and memory is freed all at once in its destructor. Read about [arena allocators here](https://en.wikipedia.org/wiki/Region-based_memory_management), or have a look at [Zig's implementation](https://github.com/ziglang/zig/blob/master/lib/std/heap/arena_allocator.zig) of one.

Using linked lists is often discouraged for performance reasons. It's valid recommendation if the nodes are allocated using a general-purpose allocator, such as malloc/free or new/delete. In these cases, the memory of each node is likely to be in completely different locations and therefore when iterating through the list, the CPU is not able to effectively cache or prefetch contiguous memory. However, if the nodes of a linked list are allocated using an arena allocator then they are most often near-contiguous and so the cache and prefetch systems of the CPU are effective.

Back to the implementation of this atomic linked list. Next, let's define what the Node structure looks like:


```cpp
// Nodes are never destroyed or freed until this class is destroyed so use-after-free is not an issue. To
// get around the issues of using-after-destructor, we use weak reference counting involving a bit flag.
struct Node {
    // reader
    ValueType *TryRetain() {
        const auto r = reader_uses.FetchAdd(1, MemoryOrder::Relaxed);
        if (r & k_dead_bit) [[unlikely]] {
            reader_uses.FetchSub(1, MemoryOrder::Relaxed);
            return nullptr;
        }
        return &value;
    }

    // reader, if TryRetain() returned non-null
    void Release() {
        const auto r = reader_uses.FetchSub(1, MemoryOrder::Relaxed);
        ASSERT(r != 0);
    }

    // Presence of this bit signifies that this node should not be read. However, increment and decrement operations
    // will still work fine regardless of whether it is set - there will be 31-bits of data that track
    // changes. Doing it this way moves the more expensive operations onto the writer thread rather than
    // the reader thread. The writer thread does atomic bitwise-AND (which is sometimes a CAS loop in
    // implementation), but the reader thread can do an atomic increment and then check the bit on the
    // result, non-atomically. The alternative might be to get the reader thread to do an atomic CAS to
    // determine if reader_uses is zero, and only increment it if its not, but this is likely more
    // expensive.
    static constexpr u32 k_dead_bit = 1u << 31;

    Atomic<u32> reader_uses;
    ValueType value;
    Atomic<Node *> next;
    Node *writer_next;
};
```

From the 2 snippets above we can begin to see how these structs might be used:
- Once allocated, Nodes are always a valid memory location.
- The reader-thread can access the live_list of nodes, but it must acquire access to the value by doing TryRetain(), and afterwards, Release(). This is like std::weak_ptr::lock.
- The writer-thread can move nodes from the live_list into the dead_list, and subsequently when no readers are using the node, it can destroy the node->value and add it to the free_list, ready to be used again.

Here's what the reader-thread can use:

```cpp
struct Iterator {
    friend bool operator==(const Iterator &a, const Iterator &b) { return a.node == b.node; };
    friend bool operator!=(const Iterator &a, const Iterator &b) { return a.node != b.node; };
    Node &operator*() const { return *node; }
    Node *operator->() { return node; }
    Iterator &operator++() {
        prev = node;
        node = node->next.Load(MemoryOrder::Relaxed);
        return *this;
    }
    Node *node {};
    Node *prev {};
};

// reader or writer
// If you are the reader the values should be considered weak references; you MUST call TryRetain (and
// afterwards Release) on the object before using it.
Iterator begin() const { return Iterator(live_list.Load(MemoryOrder::Relaxed), nullptr); }
Iterator end() const { return Iterator(nullptr, nullptr); }
```

And the features that the writer-thread can use:

```cpp
// writer, call placement-new on node->value
Node *AllocateUninitialised() {
    if (free_list) {
        auto node = free_list;
        free_list = free_list->writer_next;
        ASSERT(node->reader_uses.Load() & Node::k_dead_bit);
        return node;
    }

    auto node = arena.NewUninitialised<Node>();
    node->reader_uses.Raw() = 0;
    return node;
}

// writer, only pass a node just acquired from AllocateUnitialised and placement-new'ed
void DiscardAllocatedInitialised(Node *node) {
    node->value.~ValueType();
    node->writer_next = free_list;
    free_list = node;
}

// writer, node from AllocateUninitalised
void Insert(Node *node) {
    // insert so the memory is sequential for better cache locality
    Node *insert_after {};
    {
        Node *prev {};
        for (auto n = live_list.Load(MemoryOrder::Relaxed); n != nullptr;
                n = n->next.Load(MemoryOrder::Relaxed)) {
            if (n > node) {
                insert_after = prev;
                break;
            }
            prev = n;
        }
    }

    // put it into the live list
    if (insert_after) {
        node->next.Store(insert_after->next.Load());
        insert_after->next.Store(node);
    } else {
        node->next.Store(live_list.Load());
        live_list.Store(node);
    }

    // signal that the reader can now use this node
    node->reader_uses.FetchAnd(~Node::k_dead_bit);
}

// writer, returns next iterator (i.e. instead of ++it in a loop)
Iterator Remove(Iterator iterator) {
    if constexpr (DEBUG_CHECKS_ENABLED) {
        bool found = false;
        for (auto n = live_list.Load(MemoryOrder::Relaxed); n != nullptr;
                n = n->next.Load(MemoryOrder::Relaxed)) {
            if (n == iterator.node) {
                found = true;
                break;
            }
        }
        ASSERT(found);
    }

    // remove it from the live_list
    if (iterator.prev)
        iterator.prev->next.Store(iterator.node->next.Load());
    else
        live_list.Store(iterator.node->next.Load());

    // add it to the dead list. we use a separate 'next' variable for this because the reader still might
    // be using the node and it needs to know how to correctly iterate through the list list rather than
    // suddendly being redirecting into iterating the dead list
    iterator.node->writer_next = dead_list;
    dead_list = iterator.node;

    // signal that the reader should no longer user this node
    iterator.node->reader_uses.FetchAdd(Node::k_dead_bit);

    return Iterator {.node = iterator.node->next.Load(), .prev = iterator.prev};
}

// writer
void Remove(Node *node) {
    Node *previous {};
    for (auto it = begin(); it != end(); ++it) {
        if (it.node == node) break;
        previous = it.node;
    }
    Remove(Iterator {node, previous});
}

// writer
void RemoveAll() {
    for (auto it = begin(); it != end();)
        it = Remove(it);
}

// writer, call this regularly
void DeleteRemovedAndUnreferenced() {
    Node *previous = nullptr;
    for (auto i = dead_list; i != nullptr;) {
        ASSERT(i->writer_next != i);
        ASSERT(previous != i);
        if (previous) ASSERT(previous != i->writer_next);

        if (i->reader_uses.Load() == Node::k_dead_bit) {
            if (!previous)
                dead_list = i->writer_next;
            else
                previous->writer_next = i->writer_next;
            auto next = i->writer_next;
            i->value.~ValueType();
            i->writer_next = free_list;
            free_list = i;
            i = next;
        } else {
            previous = i;
            i = i->writer_next;
        }
    }
}
```

There's a slightly strange pattern in the above snippet where adding items to the list is a 3-step process. First you must call AllocateUninitialised(), then placement-new the node->value and finally pass it to Insert(). This could certainly be combined into a single operation. However, in my case a placement-new is necessary when the ValueType is non-copyable and non-moveable.

The writer-thread is probably a thread that always runs in 'background' of your application. It is probably some sort of event-loop that wakes-up to respond to requests. In my case, the writer-thread wakes-up when it is informed of changes to sample-library configuration files. It then reads and decodes files and makes changes to the AtomicRefList, and finally calls the DeleteRemovedAndUnreferenced() method to clean up any unused items.

I hope that this is interesting or helpful to someone. Please leave any suggestions or comments below.
