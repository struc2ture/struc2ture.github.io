---
layout: page
name: Mnemosyne. Relocatable Memory & Writing a Malloc.
title: null
---

# Mnemosyne. Relocatable Memory & Writing a Malloc.

_July 1, 2026_

There are two parts to this story: a prologue and an epilogue. One, an experiment with relocatable memory. Two, a learning project to implement my own malloc. The result is an experimental library, Mnemosyne.

## One. Relocatable Memory & Relative Pointers

When working on games I've often questioned why it's so complicated to implement even the simplest version of save files. Immediately you need to bring in complex serialization libraries or hand roll your own reflection to traverse the state and write the right data to disk. Then there's versioning. Of course you will need these things for a mature project. But why do you have to bear this complexity upfront, even in the early stages of development, when it's most important for the codebase to not limit experimentation and exploration?

In fact, backwards compatibility of save files is most important after release on the game, not when it's in the middle of development. While versioning, which you need to support backwards compatibility, is most difficult in the early and middle stages of a project, because things are in constant flux, not when the project is nearing release. So how does serialization and save files turn into this baggage that you have to carry throughout the whole development cycle, when it's a minor feature to make your project production ready?

Of course, saving and restoring the state of the application is very helpful during development too, but maybe there's a transparent way to achieve that - along the lines that emulators can snapshot and reload the whole state of a game? In fact, this can even serve as a simple save mechanism for small games and prototypes. For example, for a game jam, where all you need is to provide the player with the convenience of quitting and reopening the game on the same machine.

I am familiar with techniques of preallocating memory and using memory arenas popularized (or at least made known to me) by Casey Muratori, but it was not the complete answer for me, because all memory addressing was still done using absolute pointers, so when a program is restarted, all those addresses are unusable. I understand that the Handmade way is probably the most practical, the best of both worlds; and you could always turn off ASLR when you compile, but then you could never "ship" this as even a provisional solution in a prototype (e.g. in the game jam example above). So in the spirit of "reinventing the wheel", I continued with this experiment.

---

So, to make a memory blob truly relocatable with no modifications, all pointers inside of that blob have to be relative to its base address. But having to know the base address at every site of usage of a pointer seems very impractical. You also lose type information if such an offset is just a number. In addition, it seems very error-prone if you have multiple such base addresses. 

We need to make the base address available globally, and we need to make the compiler aware of the type of each relative pointer. There's one factor that is to our advantage here: the fact that the persistent state of an application is by definition singular. So we can know at compile time what base a relative pointer refers to: "whatever the base address associated with the persistent state is." And, of course, we can bake the type of a pointer into our pointer structure using C++ templates for example. In this way, we can have all the same compile-time guarantees that we would have with a regular pointer.

This is what I came up with when experimenting with this idea in my projects. I created `RelPtr<ValueT>` that could be dereferenced only with a given base address:

```c++
ValueT *RelPtr::get_ptr(void *base_address)
```

Then I made a special pointer type that is deliberately hardcoded for the persistent state: `StatePtr`. This one could be dereferenced without knowing the base address:

```c++
template <typename T>
struct StatePtr
{
    RelPtr<T> rel_ptr;
    
    T *operator->()
    {
        return rel_ptr.get_ptr(_state_arena_mem);
    }
    
    // ...
}
```

This allowed for usage like this:

```c++
struct State
{
    StatePtr<float> some_numbers;
}

void app_update(void *mem)
{
    bool is_initialized;
    state_arena_attach(mem, &is_initialized);
    StatePtr<State> state = state_arena_get_state_ptr();
    
    if (!is_initialized)
    {
        state->some_numbers = state_arena_push<float>(10);
        for (int i = 0; i < 10; i++)
        {
            state->some_numbers[i] = rand_float();
        } 
    }
    
    for (int i = 0; i < 10; i++)
    {
        draw_string("%d: %f\n", i, state->some_numbers[i]);
    }
}
```

This seems like a nice, non-intrusive API for something like this, but there are a couple of problems. For one, what if we have multiple arena contexts? Do we have to reimplement a structure like `StatePtr` for every separate arena context, including all the overloaded operators, etc.? It would make sense to make the fact that a pointer refers to a specific arena itself templated. In addition, the whole construct of `StatePtr<State>` for the root struct seems redundant.

Thinking of the notion of the root struct further, I came to the realization that it's almost like a header for all the allocations on the arena. It is the thing that describes the layout of, and the way to get to, everything on the arena. So I formalized it: a relative pointer is now also generic with respect to the arena scope, or "root struct", that it belongs to. This is what we can do with the updated API:

```c++
struct State
{
    RPtr<State, float> some_numbers;
}

void app_update(void *mem)
{
    bool is_initialized;
    State &state = arena_attach<State>(mem, &is_initialized);
    
    if (!is_initialized)
    {
        state.some_numbers = arena_alloc<State, float>(10);
        for (int i = 0; i < 10; i++)
        {
            state.some_numbers[i] = rand_float();
        } 
    }
    
    for (int i = 0; i < 10; i++)
    {
        draw_string("%d: %f\n", i, state->some_numbers[i]);
    }
}
```

Now, every relative pointer can be thought of as being in the context of its root struct. `RPtr<State, ValueT>` means a pointer to something that belongs to the struct State, but is not in-place in that struct at compile time. The root struct itself can be returned as a regular pointer or a C++ reference. The only restriction is that that pointer cannot be saved and accessed across different frames.

All meta data for an arena and its allocations are in-place in the same blob of memory. RootStruct can be located without its offset being stored dynamically: it's just offset by the size of the ArenaHeader. The layout of memory in that blob is:

```
| ArenaHeader | RootStruct | Dynamic Allocations ... |
```

The most interesting thing about dealing with memory in such a way is that we can treat the persistent state of an application as always already being "offline". If you map the memory block used by the arena to a file, you can close the program and then restart it right where it left off - quite literally, not through serialization-deserialization steps that run behind the scenes.

Another thing that this opens up is that we can run multiple programs operating on the exact same state, even in parallel, given they access it in a synchronized way. This is grounds for a whole other set of experiments that I will discuss in future writeups.

## Two. Writing My Own Malloc

In many projects of mine, I find stack-based allocation more than sufficient: whenever you need more memory, just push it onto the arena. You cannot deallocate, you can only discard the whole arena. Memory reuse can be implemented on a higher level (as in, not as a feature of the allocator), e.g. using free lists. And dynamically expanding lists can be implemented with a linked list of buckets, for example.

Those things make total sense for the persistent state of an application, however it's highly inconvenient when you just need some dynamic memory temporarily, for example for a function, returning some list of items. It's possible to do this by juggling a couple of scratch arenas that get reset every frame. But, to me, the problem lies in having to think of low level allocation details at all in the cases where it doesn't pertain the true persistent state of the application. It stifles experimentation and expressiveness in the early stages of development.

This is why I decided to expand the above idea with a more generic allocator, one that can free memory. Plus, writing my own malloc just seemed like a good learning opportunity.

I largely based my allocator on the one presented in [this tutorial](https://danluu.com/malloc-tutorial/) by Dan Luu. I had to adjust my implementation to not use any absolute pointers, and I implemented (basic) alignment, block splitting and coalescing. All of this works with `RPtr` as presented above, the new addition being that a relative pointer can also be freed. The end result is certainly not a production-ready malloc, but a great proof of concept and starting point.

The gist of the allocator's architecture is that it's a doubly linked list of allocation blocks laid out sequentially in a continuous memory blob. Each block starts with a header that contains meta data about this block and how to traverse the list of blocks forward and backwards. This is what that meta data layout looks like for me:

```c++
struct BlockHeader
{
    size_t size;
    size_t prev_size;
    bool is_free;
    int magic; // for debugging how a block got into its current state
};
```

To get to the first block of the list, we can add the size of the `ArenaHeader` and `RootStruct` to the base address of the arena, mentioned in part one, plus any alignment restrictions.

The next block in the list is:

```c++
block_ptr + sizeof(BlockHeader) + block_ptr->size
```

The previous block is:

```c++
block_ptr - block_ptr->prev_size - sizeof(BlockHeader)
```

For an allocation we traverse the list of blocks, looking for one that is marked as free and is big enough for the allocation. If there's no such block, we create another one by pushing the requested size, plus the size of the `BlockHeader` onto the arena. If there's a free block, we reuse it.

When reusing a block, if it's big enough for another allocation (the threshold for "big enough" can be tweaked), we reduce the reused block's size to only fit the requested size, and split off its tail into another free block.

When freeing a block, it is marked as free, but if previous and following blocks are also free, we coalesce them into one block. If the following block is free, the size of the current block is expanded to include the following block, thus effectively erasing the following block. (The new-next block's prev_size has to be adjusted accordingly.) If the previous block is free, its size is expanded to include the newly freed block.

As far as alignment goes, for now my solution is simple: just align all the allocations to the max alignment of a built-in type (max_align_t) in the system. I also make the actual size of each allocation a multiple of max_align_t, and keep every meta struct stored on the arena a multiple of max_align_t as well. This way, every block, and its meta data are aligned. C malloc itself aligns every allocation to start at a multiple of max_align_t. That of course does not deal with wider alignments, for example for SIMD vector-types. Implementing alignments like that is more complicated because we have to keep the current alignment in the block's meta data and realign it when the block is reused. As such, it remains an exercise for future me.

There are a lot of improvements that can be done to this allocation schema, the most fundamental of which is classifying allocations by size and dealing with each size accordingly. For example, for small allocations, include multiple size-classes, round up each allocation to its size-class and store each size-class sequentially. That way, you can find a block with pointer arithmetic, without O(n) list traversal. For large allocations, you can use pages, a multiple of which is used for an allocation. A great resource for a more advanced allocator design is [this TCMalloc overview](https://goog-perftools.sourceforge.net/doc/tcmalloc.html).

## Mnemosyne

The result of my endeavours is this experimental library, [Mnemosyne](https://github.com/struc2ture/mnemosyne). Check it out! (But don't use it in a real project!) There's more explanation about the API and the implementation in the repo's readme, and of course the source code is available, but here's a sample of what using that library looks like:

```c++
#include "mnemosyne.h"

void module_run_one(void *mem)
{
    bool is_initialized;
    State &state = mne_attach_mem_owning<State>(mem, is_initialized);

    if (!is_initialized)
    {
        state.numbers = mne_alloc<State, int>(10);
        mne_free(state.numbers);
        state.numbers = mne_alloc<State, int>(20);
        for (int i = 0; i < 20; i++) { state.numbers[i] = i + 1; }
    }
}

void module_run_two(void *mem)
{
    bool attahed = mne_attach_mem_non_owning<State>(new_mem);
    if (attached)
    {
        State &state = mne_get_root_struct<State>();
        draw_numbers(state.numbers, 20);
    }
}

int main()
{
    size_t mem_size = MB(4);
    void *mem = reserve_mem(mem_size);

    module_run_one(mem);

    mne_detach_mem<State>();

    void *new_mem = reserve_mem(mem_size);
    memcpy(new_mem, mem, mem_size);
    free_mem(mem, mem_size);

    module_run_two(new_mem);

    reutrn 0;
}
```

The two functions, `module_run_one` and `module_run_two`, can be run in the same process as above, but, most interestingly, they can be considered two entry points into two completely separate programs, run in separate processes: as long as data in the memory blob, wherever it is, is the same, and as long as both programs know the layout of the RootStruct (and it didn't change), both programs will execute correctly. This is what I mean by the notion of the persistent state being always already offline. I think it opens up very interesting possibilities.
